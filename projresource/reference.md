# Technical Reference — SHPF

## Architecture

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

All services communicate via Redis Streams. Each service is a standalone Python process/container.
Build and test one box at a time; never couple two boxes' internals directly.

---

## Data Contracts

### FailureEvent
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

### RootCause
```json
{
  "event_id": "uuid4 (matches FailureEvent)",
  "root_cause": "same enum as failure_type, or 'unknown'",
  "confidence": "float 0-1",
  "classifier_used": "rule_engine | ml_fallback",
  "explanation": "string, human-readable reasoning"
}
```

### PatchProposal
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

### AuditRecord (Postgres table `audit_log`)
Columns: `id, event_id, proposal_id, failure_type, root_cause, confidence, approver, decision, applied_at, rolled_back_at, notes`.

---

## Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Orchestration | Apache Airflow 2.9+ | LocalExecutor, not CeleryExecutor |
| Warehouse (dev) | PostgreSQL 16 | Stands in for Snowflake |
| Data quality checks | Great Expectations 0.18+ | |
| Drift detection | Evidently AI (OSS) | |
| Schema diffing | jsonschema + deepdiff | Custom, lightweight |
| Root cause engine | Custom rule engine (Python) + scikit-learn fallback | |
| Patch templates | Jinja2 | |
| Version control | GitPython + local git repo | Patch-as-commit |
| Approval UI | Streamlit | |
| Event bus | Redis Streams (OSS Redis) | |
| Metrics | Prometheus + Grafana | |
| Testing | pytest | |
| Containerization | Docker Compose | One-command local spin-up |

**No paid API, no cloud account, no proprietary SaaS required.**
Everything must run with `docker compose up`.

---

## Pipeline Config Repo

In Phase 6, the "pipeline config repo" is a **separate local git repo** (not a subfolder of `shpf/`).
This keeps the diff/rollback story clean and realistic.
