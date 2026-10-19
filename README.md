# Enablement Call Signal Scoring

> LLM-powered evaluation system that automatically scores sales call transcripts against enablement frameworks (Command of the Message, MEDDPICC, Product Acumen) across all GTM roles, replacing manual call review with scalable AI-driven coaching feedback.

![SQL](https://img.shields.io/badge/Databricks_SQL-AI_Functions-blue)
![Claude](https://img.shields.io/badge/Claude_Opus-LLM_Engine-orange)
![Prompt Engineering](https://img.shields.io/badge/Prompt_Engineering-Structured_Output-purple)
![Databricks](https://img.shields.io/badge/Databricks-Azure-red)

---

## Problem Statement

Manual call review doesn’t scale. Sales enablement teams can listen to a handful of calls per week, but organizations run hundreds. This project builds an automated scoring system that:

* Evaluates every external sales call against **12 weighted categories** spanning 3 enablement frameworks
* Scores **each individual rep** on the call (not just the organizer) — TMs, SEs, CSMs, and SDRs each get their own scorecard
* Returns **structured JSON** with scores, anchor bands, verbatim evidence, and improvement suggestions
* Supports **multi-language transcripts** (English, Spanish, French validated)

---

## Architecture

```text
┌─────────────────────────┐     ┌───────────────────────┐
│  Clari Copilot         │     │  Salesforce (SFDC)     │
│  call_details           │     │  Sales_Classification  │
│  (structured transcripts│     │  (GTM role mapping)    │
│   with speaker labels)  │     │                       │
└────────────┬────────────┘     └───────────┬───────────┘
             │                            │
             └────────────┬─────────────┘
                          ▼
        ┌───────────────────────────────┐
        │  CTE: call_participants       │
        │  LATERAL VIEW EXPLODE users   │
        │  JOIN SFDC for GTM role       │
        └───────────────┬───────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  ai_query(claude-opus)        │
        │  JSON Schema (strict: true)   │
        │  temp = 0.0 (deterministic)   │
        │  One call per rep scored      │
        └───────────────┬───────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  Category Scorecard           │  from_json + LATERAL VIEW EXPLODE
        │  (1 row per category per rep) │  → raw_score, weighted_score,
        │                               │    anchor_band, evidence, improvements
        ├───────────────────────────────┤
        │  Call Summary                 │  1 row per call
        │  (total_weighted_score,        │  → overall coaching summary
        │   overall_summary)            │
        └───────────────────────────────┘
Scoring Framework: 12 Categories × 3 Enablement Frameworks
#	Category	Weight	Framework
1	Value Narrative	12%	Command of the Message
2	Product & Technical Depth	15%	Product Acumen
3	Discovery Execution	12%	CoM + MEDDPICC
4	Competitive Positioning	5%	Product Acumen
5	Risk & Objection Handling	8%	Command of the Message
6	Prescriptive Guidance	8%	Command of the Message
7	Success Criteria	8%	MEDDPICC
8	Decision Process	5%	MEDDPICC
9	Multi-Stakeholder Engagement	8%	MEDDPICC
10	Meeting Control	5%	General Execution
11	Next Steps & Accountability	6%	General Execution
12	Rapport & Relationship Building	8%	General Execution
Anchor Bands: GOOD (80–100), MEDIUM (50–79), POOR (0–49)

Tech Stack
Layer	Tool
AI Engine	Databricks ai_query() SQL function
LLM	Claude Opus (production), Claude Sonnet (initial pilot)
Output Enforcement	JSON Schema with strict: true
Query Language	Databricks SQL (CTEs, LATERAL VIEW EXPLODE, from_json)
Source Data	Clari Copilot call transcripts (structured with speaker labels)
Role Mapping	Salesforce Sales_Classification__c for GTM role filtering
Platform	Databricks SQL Warehouse (Photon PRO)
Prompt Engineering Highlights
8 Core Rules (embedded in every scoring prompt)
Ground in transcript — score only what the transcript contains, never infer
Verbatim evidence — every score requires a direct quote
N/A handling — categories not applicable to the call type score as null
Category independence — each category scored without reference to others
Band→score enforcement — GOOD must score 80–100, MEDIUM 50–79, POOR 0–49
Tiebreaker — when evidence is borderline, anchor to the lower band
Mixed-evidence anchoring — with worked example for partial-credit scenarios
Structured improvements — actionable coaching suggestions per category
Key Iteration Milestones
Multi-rep scoring: Changed from organizer-only to all GTM roles per call (TM, SE, CSM, SDR)
Deterministic baseline: Temperature set to 0.0 for reproducible evaluation
Transcript audit-driven calibration: Baptist Memorial audit revealed under-scoring in Multi-Stakeholder Engagement — Cat 9 descriptors refined with explicit examples
N/A normalization: SUM(weighted_score) / SUM(weight_pct WHERE scored) * 100 prevents N/A categories from suppressing scores
3-band simplification: Reduced from 4 anchor bands to 3 (GOOD/MEDIUM/POOR) for coaching clarity
Pilot Results (Deterministic Baseline — Opus temp=0)
Role	Avg Score	Reps Scored
SDR	59.0	Pilot cohort
TM (Territory Manager)	55.3	Pilot cohort
SE (Sales Engineer)	50.5	Pilot cohort
CSM	48.5	Pilot cohort
23 total GOODs across all reps and categories
Rapport (Cat 12) is the only category reliably hitting GOOD across all runs
Zero-GOOD categories: Value Narrative, Discovery, Success Criteria, Meeting Control, Multi-Stakeholder
Project Structure
enablement-call-signal-scoring/
├── notebooks/
│   ├── scoring_engine.sql           # Production scoring: CTE pipeline + ai_query
│   ├── category_scorecard.sql        # from_json + EXPLODE parsing
│   └── call_summary.sql             # Aggregated per-call output
├── prompts/
│   └── scoring_rubric_v8.md         # Full prompt with 12 categories, 8 Core Rules,
│                                    #   3-band descriptors, and worked examples
├── docs/
│   ├── change_log.md                # Version history of prompt iterations
│   ├── pilot_results.md             # Baseline analysis and role-level breakdowns
│   └── project_documentation.md     # Full project documentation
└── README.md
What I Learned
Structured output is non-negotiable for LLM pipelines: JSON Schema with strict: true eliminates parsing failures and makes downstream SQL deterministic
Prompt engineering is iterative science, not creative writing: 8+ major revisions, each driven by transcript audits comparing model output to human judgment
Temperature matters more than you think: The same prompt on Opus at temp=0.7 vs temp=0.0 produced materially different score distributions — deterministic baselines are required before comparing prompt versions
Band descriptors are the real model: The LLM is only as good as the rubric — vague descriptors like "demonstrates good discovery" produce inconsistent scores; specific behavioral anchors ("asks at least two open-ended questions about business impact") produce calibrated ones
Multi-rep scoring changes the grain and the cost: Going from 1 row/call to 1 row/call/rep tripled ai_query invocations — cost and latency planning must account for average reps per call
