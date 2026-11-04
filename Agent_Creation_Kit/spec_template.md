# Spec — {{USE_CASE_NAME}}

<!--
HOW TO USE THIS TEMPLATE
- Save as specs/{{NNN}}-{{slug}}/spec.md (e.g. specs/002-defect-hotspot/spec.md).
- The spec says WHAT and WHY. Codex writes plan.md (HOW) and tasks.md from it.
- Replace every {{PLACEHOLDER}}. Delete guidance comments when done.
- Examples marked "e.g." come from the workshop use cases — replace them with yours.
- Write "TBD" + an entry in §15 Open Questions rather than guessing. Codex is told to stop on TBDs.
- Rule of thumb: if two engineers could implement a sentence differently, it is not specific enough.
-->

| Field | Value |
|---|---|
| Spec ID | {{NNN}}-{{slug}} |
| Use case | {{#}} · {{USE_CASE_NAME}} |
| Pod / spec owner | {{POD}} / {{NAME}} |
| Business SME | {{NAME, ROLE}} |
| Status | Draft · Reviewed · Approved |
| Version / date | v0.1 · {{YYYY-MM-DD}} |

---

## 1. Problem and intent

**Problem today:** {{What is done manually today, by whom, how long it takes, what goes wrong}}
<!-- e.g. "Release manager spends ~2 days per release correlating error logs with CQ defects to find what drags MEBEF down." -->

**Intent:** {{One sentence: what the agent does and for whom}}

**Primary users:** {{role(s)}} · **Decision supported:** {{the decision this output informs}}

**Success looks like:**
- {{measurable outcome, e.g. "Top-10 MEBEF drivers for a release produced in < 5 min, matching SME's list on ≥ 8 of 10"}}
- {{…}}

## 2. Scope

**In scope**
- {{capability 1 — from the use-case description}}
- {{capability 2}}

**Out of scope / non-goals**
- {{e.g. "Creating or editing ClearQuest records" · "Ranking individual engineers" · "Final regulatory determinations"}}

**Time window / releases / systems in scope:** {{e.g. "Defects created 2025-01-01 → today", "Release R12.1 vs R12.0"}}

## 3. User stories and acceptance criteria

<!-- 3–6 stories. Every AC must be testable by the eval harness or a unit test. Number them AC-1, AC-2 … -->

**US-1** As a {{role}}, I want {{capability}} so that {{benefit}}.
- **AC-1** Given {{data/state}}, when {{action/question}}, then {{observable result with exact fields/limits}}.
- **AC-2** Given {{…}}, when {{…}}, then {{…}}.

**US-2** As a {{role}}, I want {{…}} so that {{…}}.
- **AC-3** {{…}}

<!-- e.g.
US-1 As a quality lead, I want defects grouped by component and severity so that I see where risk concentrates.
AC-1 Given defects_2025_2026.csv, when I ask for Q1-2026 counts, then I get a component × severity table whose totals equal the row count for that quarter.
AC-2 Given a component with zero S1 defects, then it appears with 0, not omitted.
-->

## 4. Data

### 4.1 Sources

| ID | Source system | File / endpoint | Access mode | Period | Rows (approx.) | Owner / approval ref |
|---|---|---|---|---|---|---|
| D1 | {{e.g. ClearQuest}} | `data/{{file}}.csv` | CSV extract · read-only API | {{…}} | {{…}} | {{name · approval mail ref}} |
| D2 | {{…}} | | | | | |

### 4.2 Data dictionary (one table per file, or link `data/data_dictionary.md`)

| Field | Type | Meaning | Allowed values / format | Example | Classification |
|---|---|---|---|---|---|
| {{defect_id}} | str | {{…}} | `CQ\d{8}` | CQ00123456 | INTERNAL |
| {{severity}} | enum | {{…}} | S1–S4 | S2 | INTERNAL |
| {{timestamp}} | datetime | {{…}} | ISO-8601 with TZ | 2026-03-14T10:22+05:30 | INTERNAL |

### 4.3 Data handling rules
- Classification levels used: PUBLIC · INTERNAL · CONFIDENTIAL. Agent may read up to: **{{level}}**.
- Sanitisation confirmed: people pseudonymised (P01…), system/serial IDs hashed, no PHI / customer / site names. Confirmed by {{name}} on {{date}}.
- Free-text fields treated as **untrusted input** (may contain injected instructions): {{list fields, e.g. description, log_excerpt, standup_notes}}.
- Retention: workshop copies deleted after {{date}}.

## 5. Business rules and metric definitions

<!-- The most important section. Every number the agent reports is defined here and computed in core/ by code, not by the LLM. -->

| ID | Metric / rule | Exact definition | Edge cases | Unit / granularity |
|---|---|---|---|---|
| R-1 | {{metric}} | {{formula}} | {{how to handle}} | {{…}} |
| R-2 | {{threshold / RAG rule}} | {{…}} | | |

<!-- Worked examples from the workshop use cases:
| R-1 | MTTR | (Investigated − Accepted) + (first of Resolved/Rejected/Duplicate − Assigned) | Reopened → sum all cycles; missing state → exclude and flag; calendar hours | days, 1 dp, per month |
| R-2 | PR review wait | first review vote/comment − published (not draft created) | bots excluded | hours, median and p75 |
| R-3 | Flaky test | ≥ 2 pass/fail flips on the same commit SHA within 14 days | retried-pass counts as flip | per test |
| R-4 | Scope creep | story points added after sprint day 1 ÷ committed points | removed items not netted | % per sprint; > 15% = Amber |
| R-5 | MEBEF | {{exact formula from reliability team}} | {{dedup window, release attribution}} | per release |
-->

**Worked example (hand-computed by SME, used as a test):**
{{e.g. "CQ00123456: Accepted 03-01 09:00 → Investigated 03-03 11:24 = 2.1 d; Assigned 03-04 → Resolved 03-10 = 6.4 d; MTTR = 8.5 d"}}

## 6. Agent design

**Orchestration pattern:** {{Supervisor | Sequential | Router | Parallel fan-out | Evaluator loop}} — reason: {{…}}

| Agent / node | Responsibility | Inputs | Outputs (typed model) | Tools it may call | LLM? |
|---|---|---|---|---|---|
| Orchestrator | {{route, aggregate, final answer}} | user request | `FinalReport` | — | yes |
| {{Domain agent 1}} | {{…}} | {{…}} | `{{Model}}` | {{tool names}} | yes / no |
| {{Metrics node}} | compute R-1…R-n deterministically | data | `{{Metrics}}` | — | **no** |

**Shared state (LangGraph):** {{fields, e.g. request, period, records_ref, metrics, findings[], citations[], approvals[]}}
**Limits:** recursion limit {{25}} · max tool calls per run {{20}} · max tokens per run {{…}} · timeout {{60 s}}
**Human-in-the-loop points:** {{e.g. "before posting summary to Teams", "before marking a finding resolved"}}

## 7. MCP tools

<!-- Default every tool to read-only. A write tool needs §10 guard + human approval. -->

| Tool | Purpose | Input schema (Pydantic) | Output schema | Side effects | Guard |
|---|---|---|---|---|---|
| `{{search_defects}}` | {{…}} | `{component?: str, severity?: S1–S4, from: date, to: date, limit ≤ 500}` | `list[DefectSummary]` | none (read) | date range ≤ 24 months |
| `{{get_state_history}}` | {{…}} | `{defect_id: CQ\d{8}}` | `list[StateChange]` | none | — |
| `{{…write tool, if any}}` | {{…}} | {{…}} | {{…}} | **writes** | business rule + auth + human approval |

Server name: `{{uc-slug}}-mcp` · Transport: stdio (local) · Tool descriptions reviewed for clarity (they are model-facing).

## 8. API and UI

**FastAPI endpoints** (request **and** response models required)

| Method · path | Request model | Response model | Notes |
|---|---|---|---|
| `POST /{{run}}` | `{{RunRequest}}{period, filters}` | `{{RunResponse}}{run_id, report, citations}` | async; returns run_id if long |
| `POST /approve/{run_id}` | `ApprovalRequest{decision, reviewer, comment}` | `ApprovalResponse` | resumes LangGraph interrupt |
| `GET /health` | — | `Health` | |

**Streamlit UI:** {{screens: e.g. filter panel · KPI tiles · chart(s) · findings table with source links · agent trace · approve/reject}}

## 9. Output contract (for the Dashboard Orchestrator)

Every use-case agent writes `outputs/uc{{#}}_{{slug}}.json` matching this shape (agreed across all pods — do not change field names):

```json
{
  "use_case": "uc{{#}}_{{slug}}",
  "generated_at": "2026-11-04T16:30:00Z",
  "period": {"from": "2026-10-01", "to": "2026-10-31", "label": "Oct-2026"},
  "kpis": [
    {"metric": "{{MTTR_days}}", "value": 8.5, "unit": "days", "trend": "up|down|flat",
     "rag": "Green|Amber|Red", "threshold_rule": "R-2", "source": "D1"}
  ],
  "top_items": [
    {"id": "{{CQ00123456}}", "title": "{{…}}", "category": "{{component/subsystem}}",
     "severity": "{{…}}", "reason": "{{why it is top}}", "source_ref": "{{record id / url}}"}
  ],
  "risks": [
    {"summary": "{{…}}", "level": "High|Medium|Low", "area": "{{component/team}}", "evidence": ["{{source_ref}}"]}
  ],
  "needs_human_review": ["{{items flagged for review}}"],
  "run_id": "{{uuid}}"
}
```

## 10. Guardrails and safety

**Azure AI Content Safety configuration**
- Prompt Shields: user-prompt attacks on input; document attacks on every free-text record field
- Harm categories (hate, sexual, violence, self-harm): block at severity ≥ {{level}}
- Blocklists / custom categories: {{e.g. "medical diagnosis or treatment advice", "final legal determination"}}
- PII detection: {{e.g. person names, emails, phone numbers → redact; custom pattern for serial numbers → block}}
- Groundedness detection: {{on/off; applies when the agent answers from retrieved sources}}
- Models used (spec §6): {{e.g. Claude Sonnet 5.5 orchestrator · Claude Haiku 5.5 classification · GPT-4.1 fallback}}

**App-level layers**
| Layer | Rule for this use case |
|---|---|
| L1 Input | {{e.g. Prompt Shields on user prompt and on every free-text field before it reaches the model}} |
| L2 Tool call | {{allowlist, arg limits, call budget, auth}} |
| L3 Output | {{strict parse, PII scan, every claim has source_ref, numbers must equal core/ results}} |
| L4 Audit / approval | {{what is logged; which actions need approval and by whom}} |

**Must refuse / must not do**
- {{e.g. "Rank individuals by productivity" → respond with team-level view}}
- {{e.g. "Declare a field issue reportable" → mark "for complaint-handling review"}}
- {{e.g. Requests outside data in scope → say data is not available}}

## 11. Non-functional requirements

| Area | Requirement |
|---|---|
| Latency | {{p95 ≤ 30 s for a standard report}} |
| Cost | {{≤ N model calls / ≤ X tokens per run}} |
| Reliability | {{retry/backoff per AGENTS.md; graceful message if an Azure model is unavailable; fallback model named}} |
| Auditability | {{run_id on every output; JSON logs per step}} |
| Reproducibility | {{same input data → same metrics (deterministic core); temperature ≤ 0.2 for summaries}} |

## 12. Evaluation (eval harness — Day 2)

**Golden set:** `evals/golden_set.jsonl` — {{10–15}} cases, SME-verified ({{name}}, {{date}}).

| Case | Question / input | Expected answer (or property) | Type | AC |
|---|---|---|---|---|
| G-01 | {{…}} | {{exact value or rubric}} | normal | AC-1 |
| G-02 | {{edge case, e.g. reopened defect}} | {{…}} | edge | AC-2 |
| G-03 | {{out-of-scope request}} | refuses with reason | refusal | — |

**Graders:** deterministic checks first (schema valid, numbers equal `core/` output, citations exist), LLM-as-judge with rubric for summaries.

**Gates (CI):**
| Metric | Threshold |
|---|---|
| Task success (golden set) | ≥ {{90}}% |
| Numeric accuracy vs core | 100% |
| Tool-call validity | ≥ {{95}}% |
| Citation coverage | 100% of claims |
| Injection block rate (red-team set) | ≥ {{95}}% |
| PII / secret leakage | 0 |

## 13. Threat model and red-team cases (Day 2)

| ID | Attack | OWASP mapping | Expected defence (layer) |
|---|---|---|---|
| T-1 | Instruction hidden in a record's free text ("ignore previous instructions and …") | LLM01 · ASI01 | L1 Prompt Shields + prompt isolation |
| T-2 | Ask agent to list individuals' performance / names | LLM02 | L3 PII filter + refusal rule |
| T-3 | Tool called with over-broad args (no date range, huge limit) | LLM06 · ASI02 | L2 arg limits |
| T-4 | Prompt to reveal system prompt or env vars | LLM07 | L3 + no secrets in context |
| T-5 | Loop / cost exhaustion | LLM10 · ASI08 | step budget + recursion limit |
| T-6 | {{use-case specific, e.g. poisoned SBOM component name, fake regulation URL}} | {{…}} | {{…}} |

## 14. Assumptions

- {{e.g. "Extract is complete for 2025–2026; no records deleted"}}
- {{e.g. "Severity values are final at extraction time"}}

## 15. Open questions

| # | Question | Owner | Needed by |
|---|---|---|---|
| Q1 | {{e.g. "Is MTTR measured in calendar or working hours?"}} | {{SME}} | {{date}} |

## 16. Hand-off to Codex

> Codex: read this spec and `AGENTS.md`. Produce `plan.md` (format in AGENTS.md §9) and stop for review. Do not write code until the plan is approved. Raise every TBD and open question before planning.

## Change log

| Version | Date | Author | Change |
|---|---|---|---|
| v0.1 | {{…}} | {{…}} | Initial draft |
