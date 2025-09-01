# How I'd Manage Product Evals at Le Chat (v0)

## Scope & Surfaces

Surfaces: 

* chat core; 
* Research reports; 
* Think traces; 
* Tools (code, image, canvas, web); 
* Connectors (Gmail, Calendar); 
* Libraries/Projects (RAG); 
* Agents (premade/custom). 

Failure modes: 

* hallucinations/ungrounded claims; 
* tool‑call failures; 
* unsafe outputs; 
* privacy/scoping errors; 
* latency spikes; 
* cost creep.

## Deliverables

### 1. Offline Evaluation (pre-deployment + nightly CI)

Run against curated datasets or replayed, anonymized logs without affecting users. Used for PR gates, nightly regression checks, and to qualify candidates before any traffic.

Runner. A small Python service reads YAML test manifests, executes tests, and emits JSONL results. Deterministic seeds, n‑retries, and confidence intervals by default.

Tests are defined in config files (YAML/TOML) so PMs and engineers can manage them without coding.

Each test defines input, target behavior, and acceptance threshold. 

Judging signals:

* Deterministic checks. Exact/regex match; unit tests for Code; citation‑presence + link checks for Research; connector “dry‑run” validations.
* Model‑graded rubrics. Relevance, faithfulness, helpfulness, refusal quality.
* Human spot‑checks: sampling policy with QA notes.

Eval datasets:
* Think‑mode math/coding/logic sets (exact answers + trace rubric).
* Research tasks with citations
* Code-execution notebooks with assertions
* Gmail/calendar connectors synthetic corpora
* Compliance / security datasets with correct and incorrect refusals

### 2. Online Evaluation Metrics

#### Global, cross‑surface:

Task Success Rate (TSR). #sessions with successful resolution ÷ #evaluated sessions. Sources: explicit user “done”, rubric‑judge on final turn, or tool‑specific success events. 

Helpfulness Score (HS). 1–5 rubric (LLM‑judge + periodic human sample). 

Hallucination Proxy Rate (HPR). unsupported_claims ÷ total_claims (claims are extracted by LLM judge; “unsupported” = not justified by retrieved sources or connectors).

Safety Flag Rate (SFR). policy_flags ÷ outputs (or tokens)

Latency (L95/L99). p95/p99 end‑to‑end and by component (model, retrieval, tool, connector).

Cost per Session (CPS). tokens×price + tool/connector compute; include retries; track moving average.

Abandonment Rate (AR). sessions ending ≤2 turns without success.

Retry/Regenerate Rate (RR). #regens ÷ #sessions. 

Thumbs Up/Down (Like/Dislike).

#### Tools:

**Code Interpreter**: unit‑test pass‑rate; sandbox error rate; timeout rate; wall‑clock time.

**Web Search**: retrieval success (docs@k found), grounded answer rate, citation integrity.

#### Connectors (example):

**Gmail**: Draft Acceptance Rate (DAR) = accepted_drafts ÷ generated_drafts

### 3. Experiments & Rollout Rules

Ladder: shadow -> canary (1-5%) -> full A/B testing

Ship rule example: 

* improve Task Success Rate by 1-2%
* no regressions on Safety Flag Rate / Hallucination Proxy Rate. 
* P95 latency <= +5%
* Cost per Session within budget

Rollback conditions: regression detected, SLO breach, safety spikes.

### 4. Observability

Use OpenTelemetry for model inference, retrieval, tool call, connector action. 

Event schema: 

```
session_id, surface, model_id, prompt_hash, system_prompt_ver, agent_id, project_id, tool, connector_action, token_in/out, latency_ms, cost_usd, safety_flags, eval_id
```
Logging. Emit structured events for: user message, candidate decisions (route, memory read/write), evaluator outputs, and safety filters.

Dashboards. Scorecards (global and per surface). 

Alerts on metric spikes and regressions. 

Anomaly detection on metrics. 

## Partner with Science

Tight loop with Science. Co‑design evals with the modeling team, especially for new model versions, system‑prompt frameworks, and improvements in Think mode. Every experiment proposal names an on‑call Scientist to accelerate iteration, and every release links to an analysis notebook.

Post‑mortem practice. Production regressions trigger a blameless review within forty‑eight hours. We collect traces, evaluator decisions, and user impact, decide whether the eval battery or SLOs need to change, and add concrete prevention steps (new tests, stricter gates, better alerts). Findings feed back into the templates so the organization learns once, fixes everywhere.

