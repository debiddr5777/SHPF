# Phase 8 — Documentation & Demo Script

## Objective
Complete documentation and a scripted demo that showcases the entire system.

## Files to create/modify
- `README.md` — comprehensive project documentation
- `demo.sh` — scripted demo

## README.md contents
- Project title and one-paragraph description
- Architecture diagram (ASCII per `reference.md` or a rendered PNG)
- Prerequisites (Docker, Python 3.11+, Git)
- Quick start (`docker compose up`)
- Usage walkthrough (step by step)
- Project structure overview
- Tech stack table
- How to run tests
- How to run the demo

## demo.sh
- Automates the full end-to-end demonstration
- Steps:
  1. Start services (`docker compose up -d`)
  2. Wait for all services to be healthy
  3. Inject each of the six failure types in sequence
  4. For each: show detection -> classification -> patch proposal
  5. Auto-approve (or prompt user) -> show apply -> show rollback
  6. Show audit log contents
  7. Print summary (mean time to patch, approval rate, etc.)
- Uses `set -e` and has colorful output
- Checks for prerequisites before running

## Definition of Done
- `demo.sh` runs cleanly from a fresh `docker compose up`
- Full loop is demonstrated for all six failure types
- Output is readable and explains what's happening at each step
- `README.md` is sufficient for a new developer to understand and run the project
