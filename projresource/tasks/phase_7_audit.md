# Phase 7 — Audit & Dashboards

## Objective
Build the audit trail database model and Grafana dashboard.

## Files to create/modify
- `audit/models.py` — SQLAlchemy model for `audit_log` table
- `audit/grafana/dashboard.json` — Grafana dashboard export

## models.py — SQLAlchemy AuditLog model

```python
class AuditLog(Base):
    __tablename__ = "audit_log"

    id = Column(Integer, primary_key=True)
    event_id = Column(String, nullable=False)
    proposal_id = Column(String, nullable=False)
    failure_type = Column(String, nullable=False)
    root_cause = Column(String, nullable=False)
    confidence = Column(Float, nullable=False)
    approver = Column(String, nullable=True)
    decision = Column(String, nullable=False)  # approved | rejected | applied | rolled_back
    applied_at = Column(DateTime, nullable=True)
    rolled_back_at = Column(DateTime, nullable=True)
    notes = Column(Text, nullable=True)
```

## Audit logging integration
- Every phase that creates/modifies state must write to `audit_log`:
  - Detection: when FailureEvent is emitted
  - Classifier: when RootCause is emitted
  - PatchGen: when PatchProposal is emitted
  - Approval: when decision is made
  - Deployment: when apply/rollback happens

## Grafana dashboard
- Exportable JSON dashboard
- Panels:
  - Failures by type (bar chart)
  - Mean time to patch (stat)
  - Approval rate (gauge)
  - Rollback rate (gauge)
  - Failure events over time (time series)
- Data source: Postgres (the `audit_log` table)
- Dashboard JSON goes in `audit/grafana/dashboard.json`

## Definition of Done
- `audit_log` table is created in Postgres (via SQLAlchemy migration or create_all)
- Every event in the full loop is recorded in the table
- Grafana dashboard loads and shows live data after a full-loop test run
