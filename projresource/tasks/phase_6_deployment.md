# Phase 6 — Deployment Controller

## Objective
On approval, apply the patch to the pipeline config repo and update Airflow config. Provide rollback capability.

## Files to create/modify
- `deployment_controller/apply.py`
- `deployment_controller/rollback.py`

## apply.py

- Subscribes to `patch_proposals` Redis stream for proposals with status "approved"
- Steps:
  1. Clone/checkout the pipeline config repo (separate local git repo)
  2. Write the patched file content (apply diff)
  3. Create git commit with the patch
  4. Update relevant Airflow Variable or DAG config
  5. Mark proposal status as "applied" -> publish updated PatchProposal to Redis
  6. Write audit record to Postgres `audit_log`

## rollback.py

- Accepts `proposal_id`
- Steps:
  1. In the pipeline config repo, `git revert HEAD` (or the specific commit)
  2. Update Airflow config back to previous state
  3. Mark proposal status as "rolled_back" -> publish to Redis
  4. Write audit record to Postgres `audit_log`

## Pipeline config repo
- Separate local git repo (not a subfolder of `shpf/`)
- Initially contains dummy pipeline config files (YAML, SQL)
- Serves as the realistic target for patch application and rollback

## Definition of Done
- Full loop test passes:
  1. Inject failure
  2. Detect -> classify -> propose
  3. Approve via UI (or programmatic trigger)
  4. Apply -> verify pipeline config repo has the change
  5. Verify Airflow config reflects the change
  6. Roll back -> verify pipeline config repo is restored
  7. Verify proposal status transitions correctly at each step
