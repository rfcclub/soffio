---
name: loomkit-intent
description: Capture problem, desired outcome, and explicit non-goals before writing any spec.
tags: [loomkit, intent, phase-1]
---

# LoomKit Intent Phase (Hermes)

## Purpose
Separate **problem** from **proposed solution** before any spec is written.

## Inputs
- User request / pain point
- Context from previous episodes (if any)

## Output Artifact
Create `intent.md` with the following structure:

```markdown
# Intent: <short-name>

## Problem
(What is broken or missing? Be specific.)

## Desired Outcome
(What does success look like? Measurable.)

## Non-Goals
(What we will **not** do. Explicit boundaries.)

## Stakeholders
(Who cares about this?)

## Constraints
(Technical, time, policy, etc.)
```

## Rules
- Do **not** propose implementation details yet.
- If user jumps to solution, gently redirect back to problem/outcome.
- Ask clarifying questions if any section is ambiguous.

## Next Phase
After intent is approved → move to `loomkit-spec`.