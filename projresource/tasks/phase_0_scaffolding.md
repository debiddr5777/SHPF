# Phase 0 — Scaffolding

## Objective
Set up the base infrastructure and directory skeleton.

## Files to create
- `shpf/docker-compose.yml` — Postgres 16, Redis OSS, Airflow 2.9+ (webserver + scheduler), Streamlit, Prometheus, Grafana
- `shpf/README.md` — placeholder
- Full directory skeleton with `__init__.py` files

## Directory skeleton
```
shpf/
├── docker-compose.yml
├── README.md
├── dags/__init__.py
├── failure_injectors/__init__.py
├── detection/__init__.py
├── classifier/
│   ├── __init__.py
│   ├── rule_engine.py
│   └── ml_fallback.py
├── patch_generator/
│   ├── __init__.py
│   ├── templates/
│   └── generator.py
├── approval_ui/__init__.py
├── deployment_controller/
│   ├── __init__.py
│   ├── apply.py
│   └── rollback.py
├── audit/
│   ├── __init__.py
│   ├── models.py
│   └── grafana/
├── shared/
│   ├── __init__.py
│   ├── schemas.py
│   └── redis_bus.py
└── tests/
    ├── __init__.py
    ├── test_detection/__init__.py
    ├── test_classifier/__init__.py
    ├── test_patch_generator/__init__.py
    └── test_end_to_end/__init__.py
```

## Docker Compose services

| Service | Image | Port |
|---|---|---|
| postgres | postgres:16 | 5432 |
| redis | redis:7-alpine | 6379 |
| airflow-webserver | apache/airflow:2.9.0 | 8080 |
| airflow-scheduler | apache/airflow:2.9.0 | - |
| streamlit | python:3.11-slim (custom) | 8501 |
| prometheus | prom/prometheus | 9090 |
| grafana | grafana/grafana | 3000 |

## Definition of Done
- `docker compose up` succeeds when run from `shpf/`
- All containers report healthy
- Directory skeleton exists with all `__init__.py` files
- Run `pytest` (no tests yet — should pass with zero tests collected)
