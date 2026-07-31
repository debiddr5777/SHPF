# AGENTS.md — Self-Healing Data Pipeline Framework (SHPF)

You are an autonomous coding agent building the SHPF project end-to-end.
All instructions, contracts, and phase definitions are in the `projresource/` directory.
Read the reference files before starting any phase.

## Core Directives

1. **Work phase by phase per `tasks.md`.** Never skip ahead or merge phases.
2. **One taxonomy entry at a time** in Phase 2 — finish schema_drift before touching null_spike.
3. **Never modify data contracts** (`reference.md`) after Phase 1.
4. **Small, single-responsibility files** — never large modules.
5. **Every module needs a test** before moving to the next phase.
6. **Hard-code simplest version first** — add ML only after rule-based path passes.
7. **No dependencies outside tech stack** (`reference.md`) without asking the user.
8. **Run `pytest` after each phase** and report pass/fail before proceeding.
9. **All services communicate via Redis Streams** — never couple boxes' internals directly.
10. **Patch proposals must be real, syntactically valid diffs** that `git apply --check` accepts.

## Workflow

- Read `tasks.md` for phase-by-phase breakdown with Definitions of Done.
- Read `reference.md` for architecture, data contracts, and tech stack.
- Read `classification_taxonomy.md` for failure types and patch templates.
- Read `testing_strategy.md` for the testing approach.
- Read `setup_guide.md` for environment setup.
- When you encounter ambiguity, hard-code the simplest deterministic version first.
- Only ask the user for:
  - Permission to add a dependency outside the approved stack.
  - Clarification on ambiguous design decisions not covered in the TRD.
  - Permission before any non-open-source or paid-service integration.
- Report progress and test results after each phase completes.
