---
name: loomkit-plan
description: Break design into bite-sized TDD tasks with exact code.
tags: [loomkit, plan, phase-4]
---

# LoomKit Plan Phase (Hermes)

## Purpose
Convert design into a sequence of small, verifiable TDD tasks.

## Output Artifact
Create `tasks.md` with this format per task:

```markdown
### Task <T-XXX>: <Title>

**Goal:** ...
**TDD Cycle:** RED → GREEN → REFACTOR

**Files to change:**
- `path/to/file.ts` (+/- lines)

**Exact code to write (RED):**
```ts
// failing test first
```

**Exact code to write (GREEN):**
```ts
// minimal implementation
```
```

## Rules
- Tasks must be small enough to finish in < 15 minutes each.
- Every task starts with a failing test.
- No task may contain implementation without its test.

## Next Phase
`loomkit-verify`