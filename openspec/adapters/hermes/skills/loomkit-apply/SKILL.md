---
name: loomkit-apply
description: Execute TDD cycle (RED → GREEN → REFACTOR) for each task in tasks.md.
tags: [loomkit, apply, tdd, phase-5]
---

# LoomKit Apply Phase (Hermes)

## Purpose
Actually write code following strict TDD for every task defined in `tasks.md`.

## Process (per task)

1. **RED** — Write the failing test first (copy exact test from tasks.md)
2. Run test → confirm it fails
3. **GREEN** — Write the minimal code to make test pass
4. Run test → confirm it passes
5. **REFACTOR** — Clean up code while keeping tests green
6. Update `.traceability.yaml` (map SC-XXX → test file)

## Rules (Iron Law)

- **Never write production code before a failing test exists.**
- If code already exists without test → delete the code, write test first.
- One task = one commit (recommended).
- Do not move to next task until current task is GREEN + refactored.

## Output

- Code changes
- Updated `.traceability.yaml`
- All tests passing for the tasks done

## Next Phase
`loomkit-verify` (after all tasks are applied)