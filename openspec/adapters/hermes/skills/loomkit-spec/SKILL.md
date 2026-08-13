---
name: loomkit-spec
description: Turn approved intent into formal requirements using WHEN/THEN scenarios.
tags: [loomkit, spec, phase-2]
---

# LoomKit Spec Phase (Hermes)

## Purpose
Convert `intent.md` into precise, testable requirements.

## Output Artifact
Create `spec.md` (or `proposal.md` + `spec.md`) containing:

### Requirement Format (RFC 2119)

```markdown
### Requirement: <ID> — <Title>

**MUST / SHOULD / MAY** ...

#### Scenario: <SC-XXX>
**WHEN** ...
**THEN** ...
```

## Rules
- Every requirement must have at least one scenario.
- Use `MUST`, `SHOULD`, `MAY` strictly.
- No implementation details in spec.
- Traceability: Scenario IDs (`SC-001`) will be referenced in tests later.

## Next Phase
After spec passes review → `loomkit-design`.