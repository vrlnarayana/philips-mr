# AGENTS.md — {{USE_CASE_NAME}}

<!--
HOW TO USE THIS TEMPLATE
1. Copy to the ROOT of your pod repo as `AGENTS.md` (exact name — Codex reads it automatically).
2. Replace every {{PLACEHOLDER}}. Delete these HTML comments when done.
3. Keep it short and imperative. This file is for the coding agent, not for humans.
   Product intent lives in specs/<NNN>-<slug>/spec.md — do not duplicate it here.
4. Optional: put a smaller AGENTS.md inside a sub-folder (e.g. mcp_server/) to add
   folder-specific rules. The closest AGENTS.md to the file being edited wins.
-->

## 1. Project context

- **Use case:** {{USE_CASE_NAME}} — {{ONE_LINE_PURPOSE}}
- **Pod:** {{POD_NAME}} · Spec owner: {{NAME}} · Builder: {{NAME}} · Reviewer: {{NAME}}
- **Active spec:** `specs/{{NNN}}-{{slug}}/spec.md` ← the single source of truth. Read it before every task.
- **Environment:** Philips MR Agentic AI Workshop · AWS Bedrock (region `{{AWS_REGION}}`) · Bedrock Guardrails · synthetic / sanitised data only.

## 2. How you must work (spec-driven loop)

Follow this loop. Do not skip steps, do not batch tasks.

1. **Read** `spec.md` in full. If anything required for the current task is ambiguous or missing, **stop and ask** — list the open questions; do not invent business rules, thresholds or formulas.
2. **Plan** — if `plan.md` does not exist, create it using the format in §9. Wait for human approval of the plan before writing code.
3. **Tasks** — if `tasks.md` does not exist, derive it from the plan using the format in §9. Each task is small (≤ ~200 changed lines), ordered, and has a testable done-criterion.
4. **Implement exactly one task** from `tasks.md`. Touch only files that task needs.
5. **Verify** — run `make check` (lint + type check + unit tests). Fix until green. For tasks touching agent behaviour also run `make eval`.
6. **Report** — summarise what changed, which acceptance criteria (AC-x) it covers, and test results. Tick the task in `tasks.md`.
7. Stop and wait for review before the next task.

If the spec and the code disagree, the **spec wins** — raise it; do not silently "fix" the spec.

## 3. Commands

| Purpose | Command |
|---|---|
| Install / sync deps | `uv sync` |
| All checks (run before every hand-back) | `make check` → `ruff check . && ruff format --check . && mypy app core adapters mcp_server && pytest -q tests/unit` |
| Contract tests (real model, small) | `make contract` → `pytest -q tests/contract` |
| Eval harness (golden set) | `make eval` → `pytest -q evals/` (+ `npx promptfoo eval -c evals/promptfooconfig.yaml` if configured) |
| Red-team suite (Day 2) | `make redteam` → `pytest -q redteam/` |
| Run API | `uv run uvicorn app.main:app --reload --port 8000` |
| Run MCP server | `uv run python -m mcp_server.server` (stdio) · inspect: `npx @modelcontextprotocol/inspector uv run python -m mcp_server.server` |
| Run UI | `uv run streamlit run ui/app.py` |
| Environment check | `uv run python scripts/env_check.py` |

## 4. Repository layout (hexagonal — dependencies point inward)

```
specs/{{NNN}}-{{slug}}/  spec.md · plan.md · tasks.md
core/          # PURE domain: models, use cases, metric calculators, ports (Protocols). No SDK/SQL/FastAPI imports.
adapters/      # Implementations of ports: bedrock_llm.py, guardrails.py, repo_sqlite.py, mcp_client.py
agents/        # LangGraph graph, state, nodes (orchestrator + domain agents), prompts/
mcp_server/    # FastMCP server exposing this use case's tools
app/           # FastAPI airlock: routes, request/response models, DI wiring
ui/            # Streamlit front end (talks to app/ over HTTP only)
data/          # Sanitised sample extracts + data_dictionary.md (never real PHI)
tests/unit/    # Mocked LLM, deterministic
tests/contract/# Schema/property tests against the real model
evals/         # golden_set.jsonl, graders, thresholds.yaml
redteam/       # Attack scripts (Day 2)
outputs/       # Agent output files following the output contract (spec §9)
```

`core/` must never import from `adapters/`, `agents/`, `app/`, `mcp_server/` or any vendor SDK.

## 5. Non-negotiable build rules (10-point FastAPI AI scaffold)

1. Type hints on every function boundary.
2. Every LLM / tool output is parsed into a **Pydantic v2 model with `strict=True`, `extra="forbid"`**. No raw dicts flow through the code.
3. Async I/O everywhere; never call blocking I/O (`requests`, `time.sleep`, sync DB drivers) inside `async def`.
4. All model calls go through **one typed client** in `adapters/bedrock_llm.py` with timeout, retry on transient errors (exponential backoff + jitter, max 4 attempts), no retry on 4xx.
5. The model provider sits behind a **port** (`core/ports.py: LLMPort`). Bedrock is an adapter chosen at startup.
6. **Validate tool-call arguments** against the tool's Pydantic schema *before* executing; return validation errors to the model to self-correct.
7. **Destructive or write tools** need a business-rule guard + authorisation check + human approval (LangGraph `interrupt`). Default every tool to read-only.
8. Every FastAPI endpoint declares a request model **and** `response_model`. Never return raw dicts or ORM objects.
9. Tests assert **shape and properties, not wording**. Unit tests mock the LLM; contract tests hit the real model. `make check` must be green before hand-back.
10. **Pure core** — metric formulas (e.g. MTTR, cycle time, MEBEF) live in `core/` as plain, unit-tested Python. The LLM never computes a number the code can compute.

## 6. Agent and LLM rules

- **Numbers come from code, words come from the model.** The LLM summarises, classifies, explains and recommends; deterministic Python computes counts, durations, rates and rankings.
- Orchestration uses **LangGraph** with a typed state (`TypedDict` / Pydantic). Set a recursion limit and a per-run step budget. Use checkpoints.
- Prompts live in `agents/prompts/*.md`, versioned; never inline long prompts in code.
- Every answer that states a fact must carry its **source reference** (record ID, file, row, URL) — no source, no claim.
- Bedrock model ID comes from config (`BEDROCK_MODEL_ID`), never hard-coded. Use the Converse API (`ChatBedrockConverse` from `langchain-aws`, or `boto3` `converse`).
- Keep context small: pass the agent only the rows/fields it needs, never whole extracts.

## 7. Guardrails (Day 2 fills these; keep the slots from Day 1)

| Layer | Where | Must do |
|---|---|---|
| **L1 Input pre-hook** | before the orchestrator | Bedrock `ApplyGuardrail` (source=INPUT) on user text **and** on untrusted record text (descriptions, logs, notes); data-classification check |
| **L2 Tool-call guard** | inside the tool port / MCP client | schema validation, tool allowlist, business-rule + auth check, per-run call budget |
| **L3 Output validator** | before the response model | `ApplyGuardrail` (source=OUTPUT), strict model parse, PII scan, citation present |
| **L4 Audit + human gate** | at LangGraph `interrupt` | structured JSON log per step (run_id, agent, tool, args hash, result hash, guardrail action); approval for any write/notify action |

Guardrail ID/version come from `BEDROCK_GUARDRAIL_ID` / `BEDROCK_GUARDRAIL_VERSION`.

## 8. Boundaries — never do these

- Never read, print, log or commit secrets. Credentials come only from the environment / AWS profile. `.env` is git-ignored.
- Never put real PHI, patient, customer, hospital or personal names into code, prompts, tests, fixtures or logs. Use `data/` sanitised extracts only.
- Never write to external systems (Jira/ADO, ClearQuest, Trackwise, CI/CD, Teams/email). Tools are **read-only** unless the spec explicitly defines a guarded write with human approval.
- Never rank or judge individual people (developers, reviewers, engineers). Report at team / module / component level.
- Never present regulatory, complaint-reportability or safety conclusions as final — mark them "for human review".
- Never add a dependency without stating why in the task report; pin versions in `pyproject.toml`.
- Never disable a test, guardrail, type check or lint rule to make a check pass.
- Never modify `spec.md` — propose changes to the spec owner instead.

## 9. Formats for plan.md and tasks.md

**plan.md**
```
# Plan — {{USE_CASE_NAME}}
## Architecture summary        (components, data flow, which agents, which MCP tools)
## Decisions                   (choice · alternatives · reason) — one line each
## Data handling               (sources, loading, classification, sanitisation checks)
## Guardrail placement         (L1–L4 mapped to files)
## Test & eval approach        (unit, contract, golden set, thresholds)
## Risks & open questions
```

**tasks.md**
```
- [ ] T01 — <imperative title>  | covers: AC-1, AC-2 | files: core/metrics.py, tests/unit/test_metrics.py
      done when: <observable check, e.g. "pytest tests/unit/test_metrics.py passes; MTTR for sample = 8.5 d">
```
Order: scaffold → core models & metrics → adapters → MCP tools → agents/graph → API → UI → evals. One task = one reviewable diff.

## 10. Coding conventions

- Python 3.11+, `ruff` for lint/format, `mypy --strict` on `core/`.
- Names: snake_case modules/functions, PascalCase models; metric functions named after the spec (`compute_mttr`, `review_wait_hours`).
- Timestamps: timezone-aware `datetime`, stored/returned as ISO-8601 UTC.
- Logging: `structlog` JSON lines; never log full prompts or record free text at INFO.
- Commits: `T07: add defect hotspot scoring (AC-4)` — task ID first.

## 11. Use-case specific notes

<!-- Add only what Codex needs that is NOT already in spec.md. Examples:
- MTTR is computed in core/metrics.py from data/defect_state_history.csv; see spec §5.2 for edge cases.
- The agent's output must also be written to outputs/uc2_defects.json (spec §9) for the Dashboard Orchestrator.
- ClearQuest IDs look like CQ00123456; Trackwise IDs like TW-2026-00871.
-->
- {{NOTE_1}}
- {{NOTE_2}}

## 12. Definition of done (per task)

- [ ] Only the files listed in the task changed
- [ ] `make check` green; `make eval` green if agent behaviour changed
- [ ] Acceptance criteria referenced (AC-x) and demonstrably met
- [ ] No secrets, no real personal/patient data, no new unpinned dependency
- [ ] Task ticked in `tasks.md` with a 2–3 line summary
