# Failure Taxonomy & Patch Templates — SHPF

## MVP Set — Six Failure Types

| Failure type | Detection method | Patch type | Patch template output |
|---|---|---|---|
| Schema drift | Compare incoming JSON schema vs. last-known-good (deepdiff) | `schema_mapping` | Updated schema mapping YAML with new/removed columns flagged |
| Null-rate spike | Great Expectations `expect_column_values_to_not_be_null` + threshold | `null_handling` | Proposed null-handling rule (default value or filter clause) |
| Volume anomaly | Evidently row-count drift vs. rolling 7-day baseline | `volume_backfill` | Proposed backfill DAG trigger + adjusted expectation threshold |
| Duplicate spike | Row-hash count vs. baseline | `dedup_rule` | Proposed dedup SQL/dbt snippet |
| Referential-integrity break | Foreign-key sample check against dimension table | `referential_fix` | Proposed join-key remap or quarantine rule |
| Upstream API contract change | Response schema diff on mocked API source | `api_adapter` | Proposed updated parser/adapter code stub |

## Per-Type Requirements

Each of the six needs:
1. A synthetic failure injector (for reliable demos)
2. A detector
3. A classifier rule
4. A Jinja2 patch template
5. A test proving the patch resolves the injected failure

## Implementation Order

1. `schema_drift`
2. `null_spike`
3. `volume_anomaly`
4. `duplicate_spike`
5. `referential_break`
6. `api_contract_change`

Build one at a time. Never in parallel.
