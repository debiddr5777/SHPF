# Phase 5 — Approval UI

## Objective
Build the Streamlit approval UI for reviewing and approving/rejecting patch proposals.

## Files to create/modify
- `approval_ui/app.py` — Streamlit application

## Streamlit app features

1. **Proposal list** — all PatchProposals with status "pending"
2. **Proposal detail** — when a proposal is clicked:
   - Diff viewer (side-by-side or unified)
   - Explanation text
   - Confidence score
   - Target file path
3. **Action buttons**:
   - **Approve** — sets status to "approved", writes to Postgres audit_log
   - **Reject** — sets status to "rejected", prompts for notes/reason
   - **Edit** — allows inline diff editing before approval
4. **History view** — list of all proposals with their current status

## Integration
- Read from Redis stream `patch_proposals` (including historical)
- Write decisions back to Redis stream and Postgres `audit_log` table
- Use Streamlit's session state for temporary in-memory state

## Definition of Done
- UI loads and shows pending proposals
- Approve button flips status to "approved" and writes to Postgres
- Reject button flips status to "rejected" with notes
- Edit button allows editing diff before approval
- No external SaaS dependency — everything runs locally in Docker
