---
name: loomkit-verify
description: Run coverage gate and verify all scenarios are tested.
tags: [loomkit, verify, phase-5]
---

# LoomKit Verify Phase (Hermes)

## Purpose
Enforce quality gate before archiving.

## Output Artifact
Generate `.loomkit-verify.json` with:

- Coverage report
- Scenario traceability matrix
- Pass/fail status per `SC-XXX`

## Rules
- All `MUST` scenarios must have passing tests.
- Minimum coverage threshold defined in `config.yaml`.
- If gate fails → do not proceed to archive.

## Next Phase
`loomkit-archive` (only if verify passes)