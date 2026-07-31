# Build Phases — SHPF

Execute strictly in this order. Each phase has a Definition of Done (DoD).
Do not start a phase before the previous one's DoD is met and tests pass.

---

## Phase 0 — Scaffolding

**Files to create:**
- `shpf/docker-compose.yml` — Postgres 16, Redis OSS, Airflow 2.9+ (webserver + scheduler), Streamlit
- `shpf/README.md` — placeholder
- `shpf/` directory skeleton per repo structure below

**Repo structure to create:**
```
shpf/
├── docker-compose.yml
├── README.md
├── dags/
├── failure_injectors/
├── detection/
├── classifier/
│   ├── rule_engine.py
│   └── ml_fallback.py
├── patch_generator/
│   ├── templates/
│   └── generator.py
├── approval_ui/
├── deployment_controller/
│   ├── apply.py
│   └── rollback.py
├── audit/
│   ├── models.py
│   └── grafana/
├── shared/
│   ├── schemas.py
│   └── redis_bus.py
└── tests/
    ├── test_detection/
    ├── test_classifier/
    ├── test_patch_generator/
    └── test_end_to_end/
```

**DoD:**
- `docker compose up` succeeds, all containers healthy
- Directory skeleton exists with empty `__init__.py` files

---

## Phase 1 — Contracts

**Files to create/modify:**
- `shpf/shared/schemas.py` — pydantic models: `FailureEvent`, `RootCause`, `PatchProposal`
- `shpf/shared/redis_bus.py` — publish/subscribe helpers using Redis Streams
- `shpf/tests/test_shared/` — unit tests

**Data contracts (exact):**
See `reference.md` §Data Contracts. Implement these exactly — they are the seams between modules and must never change after this phase.

**DoD:**
- Unit tests publish and consume a dummy message end-to-end via Redis
- All pydantic models validate against example JSON from `reference.md`

---

## Phase 2 — Failure Injection + Detection

Build one taxonomy entry at a time (never in parallel):

1. schema_drift
2. null_spike
3. volume_anomaly
4. duplicate_spike
5. referential_break
6. api_contract_change

**Per entry:**
- `failure_injectors/<entry>.py` — synthetic failure generator
- `detection/<entry>.py` — detector that emits `FailureEvent` to Redis
- `shpf/tests/test_detection/test_<entry>.py` — test

**DoD per entry:**
- Injecting the failure produces a valid `FailureEvent` on the bus
- Verified by an automated test — not manual inspection

---

## Phase 3 — Root Cause Classifier

**Files to create/modify:**
- `classifier/rule_engine.py` — deterministic mapping from failure_type + evidence shape to RootCause
- `classifier/ml_fallback.py` — scikit-learn classifier for cases rule engine marks `unknown`
- `tests/test_classifier/`

**DoD:**
- For each of the six injected failures, classifier confidence >= 0.8 and correct root cause

---

## Phase 4 — Patch Generator

**Files to create/modify:**
- `patch_generator/templates/*.j2` — one Jinja2 template per patch_type
- `patch_generator/generator.py` — consumes RootCause, emits PatchProposal
- `tests/test_patch_generator/`

**DoD:**
- Generated diff applies cleanly with `git apply --check` in a test sandbox repo

---

## Phase 5 — Approval UI

**Files to create/modify:**
- `approval_ui/app.py` — Streamlit app
- List pending PatchProposals, diff viewer, approve/reject/edit buttons
- Writes decision back to Redis + Postgres

**DoD:**
- Manual approval flips `status` and triggers Phase 6 deployment controller

---

## Phase 6 — Deployment Controller

**Files to create/modify:**
- `deployment_controller/apply.py` — on approved, git commit + Airflow config update
- `deployment_controller/rollback.py` — revert last commit, restore prior state

**DoD:**
- Full loop test: inject failure -> detect -> classify -> propose -> approve -> apply -> verify pipeline passes -> roll back -> verify prior state restored

---

## Phase 7 — Audit & Dashboards

**Files to create/modify:**
- `audit/models.py` — SQLAlchemy model for `audit_log` table
- `audit/grafana/dashboard.json` — Grafana dashboard export

**DoD:**
- Dashboard renders live data after a full-loop test run

---

## Phase 8 — Documentation & Demo Script

**Files to create/modify:**
- `README.md` — architecture diagram, setup steps
- `demo.sh` — scripted demo injecting all six failure types in sequence

**DoD:**
- `demo.sh` runs cleanly and shows the full loop resolving each failure
