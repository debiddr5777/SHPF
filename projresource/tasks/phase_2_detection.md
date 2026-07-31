# Phase 2 — Failure Injection + Detection

## Objective
Build synthetic failure injectors and detectors for all six failure types.
Build one taxonomy entry at a time, never in parallel.

## Order
1. `schema_drift`
2. `null_spike`
3. `volume_anomaly`
4. `duplicate_spike`
5. `referential_break`
6. `api_contract_change`

## Per-entry file structure

```
failure_injectors/<entry>.py   # synthetic failure generator
detection/<entry>.py           # detector that emits FailureEvent to Redis
tests/test_detection/test_<entry>.py  # automated test
```

### failure_injectors/<entry>.py
- Generate a synthetic data sample that exhibits the failure
- Return the sample + metadata about what was injected

### detection/<entry>.py
- Accept a data sample
- Run the relevant check (GE suite, Evidently comparison, deepdiff, etc.)
- If failure detected, publish a `FailureEvent` to Redis stream `failure_events`
- If no failure, return clean result (no event emitted)

### test file
- Inject the failure using the injector
- Pass the result to the detector
- Assert a valid `FailureEvent` appears on the bus
- Assert the `failure_type` field matches

## Implementation notes
- For GE checks, use Great Expectations 0.18+ programmatic API (no CLI)
- For Evidently, use the drift detection module
- For schema diffing, use `jsonschema` + `deepdiff`
- Keep each detector under 200 lines
- Hard-code thresholds; don't build auto-tuning yet

## Definition of Done (per entry)
- Injecting the failure produces a valid `FailureEvent` on the bus
- Verified by an automated test — not manual inspection
- `pytest shpf/tests/test_detection/test_<entry>.py -v` passes
