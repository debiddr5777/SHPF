# Phase 4 — Patch Generator

## Objective
Build the patch generator service that consumes RootCause and emits PatchProposal with real, syntactically valid diffs.

## Files to create/modify
- `patch_generator/templates/*.j2` — Jinja2 templates
- `patch_generator/generator.py` — main generator service
- `tests/test_patch_generator/`

## Templates

One per patch_type:

| patch_type | Template | Output example |
|---|---|---|
| `schema_mapping` | `schema_mapping.j2` | YAML mapping with new/removed columns |
| `null_handling` | `null_handling.j2` | SQL COALESCE or WHERE clause |
| `volume_backfill` | `volume_backfill.j2` | Airflow DAG config for backfill |
| `dedup_rule` | `dedup_rule.j2` | SQL snippet with ROW_NUMBER() dedup |
| `referential_fix` | `referential_fix.j2` | Join key remap YAML |
| `api_adapter` | `api_adapter.j2` | Python parser stub |

## generator.py

- Subscribes to `root_causes` Redis stream
- Selects template based on RootCause.root_cause -> patch_type mapping
- Renders template with evidence params
- Produces unified diff string (compare proposed file vs. current file)
- Publishes `PatchProposal` to `patch_proposals` Redis stream

## Patch type mapping
```
schema_drift       -> schema_mapping
null_spike         -> null_handling
volume_anomaly     -> volume_backfill
duplicate_spike    -> dedup_rule
referential_break  -> referential_fix
api_contract_change -> api_adapter
```

## Definition of Done
- Generated diff applies cleanly with `git apply --check` in a test sandbox repo
- `pytest shpf/tests/test_patch_generator/ -v` passes
