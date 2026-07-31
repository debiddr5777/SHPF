# Phase 1 — Contracts

## Objective
Implement the shared data contracts and Redis communication layer that all other modules depend on.

## Files to create/modify
- `shpf/shared/schemas.py` — pydantic models
- `shpf/shared/redis_bus.py` — publish/subscribe helpers
- `shpf/tests/test_shared/` — unit tests

## schemas.py — pydantic models

Implement these exact models:

```python
class FailureEvent(BaseModel):
    event_id: str  # uuid4
    timestamp: str  # ISO8601
    pipeline_id: str
    task_id: str
    failure_type: Literal["schema_drift", "null_spike", "volume_anomaly", "duplicate_spike", "referential_break", "api_contract_change"]
    evidence: dict  # {"metric": str, "expected": Any, "observed": Any, "threshold": Any}
    raw_sample: Optional[str] = None

class RootCause(BaseModel):
    event_id: str  # uuid4, matches FailureEvent
    root_cause: str  # same enum as failure_type, or "unknown"
    confidence: float  # 0-1
    classifier_used: Literal["rule_engine", "ml_fallback"]
    explanation: str

class PatchProposal(BaseModel):
    proposal_id: str  # uuid4
    event_id: str  # uuid4
    patch_type: Literal["schema_mapping", "null_handling", "volume_backfill", "dedup_rule", "api_adapter"]
    diff: str  # unified diff string
    target_file: str
    explanation: str
    confidence: float  # 0-1
    status: Literal["pending", "approved", "rejected", "applied", "rolled_back"]
```

## redis_bus.py — publish/subscribe helpers

- `publish(stream: str, event: BaseModel)` — serialize to JSON, push to Redis stream
- `subscribe(stream: str, group: str, consumer: str, model_class: type[BaseModel])` — blocking read from stream, deserialize to pydantic model
- Use `redis-py` library
- Stream name = class name snake_case (e.g., `failure_events`, `root_causes`, `patch_proposals`)

## Definition of Done
- Unit tests publish a `FailureEvent` to Redis and consume it via subscribe
- Unit tests publish a `RootCause` to Redis and consume it via subscribe
- Unit tests publish a `PatchProposal` to Redis and consume it via subscribe
- All pydantic models validate against example JSON from `reference.md`
- Tests pass with `pytest -v shpf/tests/test_shared/`
