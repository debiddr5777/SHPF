# TRD — Self-Healing Data Pipeline Framework (SHPF)

**Status:** Draft v1
**Owner:** Debi Prasad Mohapatra
**Target build tool:** OpenCode + DeepSeek V3 Flash (agentic coding)
**License model:** 100% open source, no paid API/service dependency

---

## 1. Problem Statement

Data pipelines fail silently or noisily in ways that current tooling (Airflow alerts, Prometheus/Grafana dashboards) only *detects* — it never *diagnoses* or *fixes*. Engineers spend hours doing the same triage repeatedly: check schema, check nulls, check row counts, patch the DAG, redeploy. This project builds a framework that:

1. Detects data/pipeline failures at a semantic level (not just "task failed").
2. Classifies the **root cause** into a known taxonomy.
3. Generates a **concrete patch proposal** (config diff, not vague advice).
4. Routes the proposal to a human for one-click approve/reject/edit.
5. Applies the approved patch and can roll it back.
6. Logs everything for audit and for training a better classifier over time.

Non-goal: this is not a fully autonomous "no human in the loop" system. Full autonomy is where 80% of published attempts fail or overreach — the defensible, buildable, honest scope is **human-approved auto-remediation**.

---

## 2. Scope

### 2.1 In scope (MVP)
- Simulated pipelines with injected, realistic failure modes (see §6).
- Detection for: schema drift, null-rate spikes, row-count/volume anomalies, duplicate spikes, referential-integrity breaks, upstream API contract changes.
- Rule-based root-cause classifier (explainable, deterministic) with an optional ML fallback for ambiguous cases.
- Template-based patch generator producing an actual diff (YAML/JSON config or SQL/dbt snippet).
- A local web approval UI (no external SaaS dependency).
- Deployment controller that applies patches to Airflow DAG configs / dbt sources and can revert.
- Full audit trail in Postgres + Grafana dashboard.

### 2.2 Out of scope (explicitly — do not implement, do not suggest expanding into)
- Fully autonomous deployment without human approval.
- Multi-cloud abstraction layer.
- Natural-language chat interface for approvals (button-click UI is sufficient).
- Support for streaming (Kafka-style) pipelines — batch only for v1.
- Any paid API (OpenAI, Snowflake, Datadog, etc.) — see §3.

---

## 3. Tech Stack (fully open source)

| Layer | Choice | Why |
|---|---|---|
| Orchestration | Apache Airflow 2.9+ | Already known to the builder; industry standard |
| Warehouse (dev) | PostgreSQL 16 | Free, local, no vendor lock-in (stands in for Snowflake) |
| Data quality checks | Great Expectations 0.18+ | Mature OSS validation framework |
| Drift detection | Evidently AI (OSS) | Statistical drift detection out of the box |
| Schema diffing | Custom (jsonschema + deepdiff) | Lightweight, no external service |
| Root cause engine | Custom rule engine (Python) + scikit-learn fallback classifier | Explainable-first design |
| Patch templates | Jinja2 | Standard, simple, testable |
| Version control of patches | GitPython + local git repo | Every patch is a real commit, fully auditable |
| Approval UI | Streamlit (or FastAPI + plain HTML/HTMX if Streamlit proves too heavy) | Zero external dependency, runs locally |
| Event bus | Redis Streams (OSS Redis) | Simple, avoids Kafka operational overhead |
| Metrics | Prometheus + Grafana | Already in builder's stack |
| Testing | pytest | Standard |
| Containerization | Docker Compose | One-command local spin-up |

No step in this project requires a paid API key, a cloud account, or a proprietary SaaS tool. Everything must run with `docker compose up`.

---

## 4. Architecture

```
                     ┌─────────────────────┐
  Simulated Sources  │  Ingestion Layer     │
  (CSV/API mocks) ──▶│  (Airflow DAG tasks) │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ Monitoring &         │
                     │ Detection Layer      │  ── emits FailureEvent (JSON)
                     │ (GE + Evidently +    │
                     │  custom checks)      │
                     └──────────┬───────────┘
                                ▼
                     ┌─────────────────────┐
                     │ Root Cause           │
                     │ Classifier Service   │  ── emits RootCause (enum + evidence)
                     └──────────┬───────────┘
                                ▼
                     ┌─────────────────────┐
                     │ Patch Generator      │
                     │ Service (Jinja2)     │  ── emits PatchProposal (diff + explanation)
                     └──────────┬───────────┘
                                ▼
                     ┌─────────────────────┐
                     │ Approval Gateway     │◀── human clicks approve/reject/edit
                     │ (Streamlit UI)       │
                     └──────────┬───────────┘
                                ▼
                     ┌─────────────────────┐
                     │ Deployment           │
                     │ Controller           │  ── git commit + Airflow variable/DAG update
                     │ (+ rollback)         │
                     └──────────┬───────────┘
                                ▼
                     ┌─────────────────────┐
                     │ Audit & Observability│  (Postgres + Grafana)
                     └─────────────────────┘
```

All services communicate via Redis Streams using the JSON schemas in §5. Each service is a standalone Python process/container — this matters for the coding agent: **build and test one box at a time**, never couple two boxes' internals directly.

---

## 5. Data Contracts (must be implemented exactly — these are the seams between modules)

### 5.1 `FailureEvent`
```json
{
  "event_id": "uuid4",
  "timestamp": "ISO8601",
  "pipeline_id": "string",
  "task_id": "string",
  "failure_type": "schema_drift | null_spike | volume_anomaly | duplicate_spike | referential_break | api_contract_change",
  "evidence": {
    "metric": "string",
    "expected": "value",
    "observed": "value",
    "threshold": "value"
  },
  "raw_sample": "optional truncated data sample for debugging"
}
```

### 5.2 `RootCause`
```json
{
  "event_id": "uuid4 (matches FailureEvent)",
  "root_cause": "same enum as failure_type, or 'unknown'",
  "confidence": "float 0-1",
  "classifier_used": "rule_engine | ml_fallback",
  "explanation": "string, human-readable reasoning"
}
```

### 5.3 `PatchProposal`
```json
{
  "proposal_id": "uuid4",
  "event_id": "uuid4",
  "patch_type": "schema_mapping | null_handling | volume_backfill | dedup_rule | api_adapter",
  "diff": "unified diff string, actual file content change",
  "target_file": "path within repo",
  "explanation": "string",
  "confidence": "float 0-1",
  "status": "pending | approved | rejected | applied | rolled_back"
}
```

### 5.4 `AuditRecord` (Postgres table `audit_log`)
Columns: `id, event_id, proposal_id, failure_type, root_cause, confidence, approver, decision, applied_at, rolled_back_at, notes`.

---

## 6. Failure Taxonomy & Patch Templates (MVP set — implement all six)

| Failure type | Detection method | Patch template output |
|---|---|---|
| Schema drift | Compare incoming JSON schema vs. last-known-good (deepdiff) | Updated schema mapping YAML with new/removed columns flagged |
| Null-rate spike | Great Expectations `expect_column_values_to_not_be_null` + threshold | Proposed null-handling rule (default value or filter clause) |
| Volume anomaly | Evidently row-count drift vs. rolling 7-day baseline | Proposed backfill DAG trigger + adjusted expectation threshold |
| Duplicate spike | Row-hash count vs. baseline | Proposed dedup SQL/dbt snippet |
| Referential-integrity break | Foreign-key sample check against dimension table | Proposed join-key remap or quarantine rule |
| Upstream API contract change | Response schema diff on mocked API source | Proposed updated parser/adapter code stub |

Each of these needs: (1) a synthetic failure injector so it can be demoed reliably, (2) a detector, (3) a classifier rule, (4) a Jinja2 patch template, (5) a test proving the patch resolves the injected failure.

---

## 7. Repository Structure (create exactly this — do not improvise a different layout)

```
shpf/
├── docker-compose.yml
├── README.md
├── dags/                        # Airflow DAGs (ingestion + pipeline DAGs)
├── failure_injectors/           # synthetic failure generators, one file per taxonomy entry
├── detection/                   # GE suites + Evidently configs + custom checks
├── classifier/
│   ├── rule_engine.py
│   └── ml_fallback.py
├── patch_generator/
│   ├── templates/               # Jinja2 templates, one per patch_type
│   └── generator.py
├── approval_ui/                 # Streamlit app
├── deployment_controller/
│   ├── apply.py
│   └── rollback.py
├── audit/
│   ├── models.py                # SQLAlchemy models
│   └── grafana/                 # dashboard JSON exports
├── shared/
│   ├── schemas.py                # pydantic models for §5 contracts
│   └── redis_bus.py
└── tests/
    ├── test_detection/
    ├── test_classifier/
    ├── test_patch_generator/
    └── test_end_to_end/
```

---

## 8. Build Phases (execute strictly in this order — each phase has a Definition of Done)

**Phase 0 — Scaffolding**
- `docker-compose.yml` brings up: Postgres, Redis, Airflow (webserver+scheduler), Streamlit.
- DoD: `docker compose up` succeeds, all containers healthy.

**Phase 1 — Contracts**
- Implement `shared/schemas.py` (pydantic models for FailureEvent, RootCause, PatchProposal) and `shared/redis_bus.py` (publish/subscribe helpers).
- DoD: unit tests publish and consume a dummy message end-to-end via Redis.

**Phase 2 — Failure Injection + Detection (one taxonomy entry at a time)**
- Start with `schema_drift` only. Build injector → detector → passing test. Then repeat for the other five, one at a time, never in parallel.
- DoD per entry: injecting the failure produces a valid `FailureEvent` on the bus, verified by an automated test — not manual inspection.

**Phase 3 — Root Cause Classifier**
- Rule engine first (deterministic mapping from `failure_type` + evidence shape to `RootCause`). ML fallback only for cases the rule engine marks `unknown` — use a small scikit-learn classifier trained on synthetic evidence vectors.
- DoD: for each of the six injected failures, classifier confidence ≥ 0.8 and correct root cause.

**Phase 4 — Patch Generator**
- One Jinja2 template per `patch_type`. Generator consumes `RootCause`, emits `PatchProposal` with a real, syntactically valid diff.
- DoD: generated diff applies cleanly with `git apply --check` in a test sandbox repo.

**Phase 5 — Approval UI**
- Streamlit app listing pending `PatchProposal`s, diff viewer, approve/reject/edit buttons, writes decision back to Redis + Postgres.
- DoD: manual approval flips `status` and triggers Phase 6.

**Phase 6 — Deployment Controller**
- On `approved`, commit the diff to a local git repo representing the "pipeline config repo," update the relevant Airflow Variable/DAG config, mark `applied`. Provide `rollback.py` reverting the last commit and reapplying the Airflow config.
- DoD: full loop test — inject failure → detect → classify → propose → approve → apply → verify pipeline now passes → roll back → verify prior state restored.

**Phase 7 — Audit & Dashboards**
- Every event/decision/apply/rollback written to `audit_log`. Grafana dashboard: failures by type, mean time-to-patch, approval rate, rollback rate.
- DoD: dashboard renders live data after a full-loop test run.

**Phase 8 — Documentation & Demo Script**
- README with architecture diagram, setup steps, and a scripted demo (`demo.sh`) that injects each of the six failure types in sequence and shows the full loop resolving them.

Do not start a phase before the previous one's DoD is met and tests pass. This bounds scope creep, which is the main risk with a fast/lightweight coding model.

---

## 9. Testing Strategy
- Unit tests per module (detection, classifier, patch generator) — no module is "done" without tests.
- Integration test per failure type (injector → bus → detector → classifier → patch, asserting on message contents, not print statements).
- One full end-to-end test (Phase 6 DoD) that is the acceptance test for the whole project.
- Target: `pytest` green across `tests/` before any phase is considered complete.

---

## 10. Instructions for the Coding Agent (OpenCode + DeepSeek V3 Flash)

- Work phase by phase per §8. Do not jump ahead or merge phases.
- Never modify the data contracts in §5 once Phase 1 is done — all later modules depend on them being stable.
- Prefer small, single-responsibility files over large modules — this model performs better with tight, isolated context per file.
- Every new module needs a corresponding test file in the same phase before moving to the next phase.
- If a detection/classification rule is ambiguous, hard-code the simplest deterministic version first; do not attempt an ML model until the rule-based path is passing.
- Do not introduce any dependency outside §3's stack without flagging it back to the user first.
- After each phase, run the full `pytest` suite and report pass/fail before proceeding.

---

## 11. Open Questions (resolve before or during Phase 0)
- Exact Airflow deployment mode for local dev: `LocalExecutor` vs. `CeleryExecutor` — recommend `LocalExecutor` for simplicity.
- Whether the "pipeline config repo" in Phase 6 is a real separate git repo or a subfolder of `shpf/` — recommend a separate local repo to keep the diff/rollback story clean and realistic.
