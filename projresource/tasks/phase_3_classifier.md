# Phase 3 — Root Cause Classifier

## Objective
Build the root cause classifier service.

## Files to create/modify
- `classifier/rule_engine.py`
- `classifier/ml_fallback.py`
- `tests/test_classifier/`

## rule_engine.py

Deterministic classifier. Consumes `FailureEvent`, produces `RootCause`.

Logic:
- Match `failure_type` from event directly — that's the root cause
- Map evidence shape to confidence:
  - If evidence clearly matches the failure_type pattern -> confidence 0.9-1.0
  - If evidence is ambiguous -> confidence 0.5-0.7
  - If evidence doesn't match known patterns -> root_cause = "unknown", confidence 0.0-0.3
- Always set `classifier_used = "rule_engine"`

Rules per failure type:
- `schema_drift`: expected/observed have different keys -> high confidence
- `null_spike`: metric = "null_rate", observed >> threshold -> high confidence
- `volume_anomaly`: metric = "row_count", observed deviates from expected by > threshold -> high confidence
- `duplicate_spike`: metric = "duplicate_rate", observed >> threshold -> high confidence
- `referential_break`: metric = "orphan_rate", observed > 0 -> high confidence
- `api_contract_change`: metric = "response_schema_change" -> high confidence

## ml_fallback.py

Optional scikit-learn classifier for cases the rule engine marks "unknown".

- Train on synthetic evidence vectors
- Small model (e.g., LogisticRegression or RandomForest with ~100 samples)
- Only invoked when rule engine returns confidence < 0.5 or root_cause == "unknown"
- Set `classifier_used = "ml_fallback"` in output

## Integration
- Top-level `classifier/__init__.py` or `classify.py` that:
  1. Subscribes to `failure_events` Redis stream
  2. Runs rule engine
  3. If rule engine confidence < 0.5, run ML fallback
  4. Publishes `RootCause` to `root_causes` Redis stream

## Definition of Done
- For each of the six injected failures, classifier confidence >= 0.8 and correct root cause
- `pytest shpf/tests/test_classifier/ -v` passes
