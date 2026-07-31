# Testing Strategy — SHPF

## Levels

1. **Unit tests** per module — detection, classifier, patch generator.
   - No module is "done" without tests.
   
2. **Integration tests** per failure type — injector -> bus -> detector -> classifier -> patch.
   - Assert on message contents, not print statements.

3. **End-to-end test** — Phase 6 DoD is the acceptance test for the whole project:
   - Inject failure -> detect -> classify -> propose -> approve -> apply -> verify pipeline passes -> roll back -> verify prior state restored.

## Target
- `pytest` green across `tests/` before any phase is considered complete.

## Conventions
- Test files go in `shpf/tests/<module>/test_<feature>.py`
- Use `pytest` fixtures for Redis, Postgres, and Airflow test instances
- Every test must be self-contained (no shared mutable state between tests)
- Integration tests should use lightweight test containers (or mocked Redis/Postgres) for CI speed
