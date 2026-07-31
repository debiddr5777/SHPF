# SHPF — Self-Healing Data Pipeline Framework

Detects data/pipeline failures at a semantic level, classifies root causes,
generates concrete patch proposals, and routes them to a human for
one-click approve/reject/edit — then applies approved patches with rollback.

> Full documentation, architecture, and demo script land in Phase 8.

## Quick start

```bash
docker compose up -d
```

| Service | URL |
|---|---|
| Airflow webserver | http://localhost:8080 (admin / admin) |
| Streamlit approval UI | http://localhost:8501 |

## Project layout

See `projresource/reference.md` for architecture and data contracts.
See `projresource/tasks.md` for the phase-by-phase build plan.
