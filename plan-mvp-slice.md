# Soffio MVP Slice — Concrete Plan

Agrees with Coda's verdict (2026-08-12, relayed via thoor): intent.md
is ready for LoomKit's Spec → Design phases, not code yet. This
document is the task breakdown + dependency graph for the vertical
slice Coda proposed, so the still-open pillars (2 self-responsibility
gate, 4 AgentRadio background listener, MVP ordering) get a real
boundary to design against instead of staying abstract.

**Scope of this slice** (Coda's framing, adopted as-is): 1 identity +
1 `WorkTask` + `Provider`/`AgentSession` with `modelOverride` + one
`colony-mesh` thread/message + evidence-gated completion. Not 16
agents, not Docker Sandboxes, not full AgentRadio background push —
those stay deferred per `intent.md`'s Non-Goals and Open Questions.

## Dependency graph

```
D0: GardenHub WorkTask ↔      D1: Self-responsibility
    colony-mesh contract          gate v0 definition
    (design decision)             (design decision)
         │                              │
         │ (independent of D1)          │ (independent of D0)
         ▼                              ▼
    Track B: colony-mesh          Track C: Gate v0
    participation                 implementation
         │                              │
Track A: Rialto minimal spawn ─────────┤ (no dependency on D0/D1,
(agent-runtime wiring)                  │  can start immediately)
         │                              │
         └──────────────┬───────────────┘
                         ▼
              Integration: wire the
              full vertical slice
                         │
                         ▼
              Verify: real end-to-end
              run + negative test
```

## D0 — GardenHub WorkTask ↔ colony-mesh contract (design decision, blocking Track B)

Not implementation — a decision. What needs answering before Track B
can be written:

- Does a `WorkTask` (`src/garden/hub.ts`) carry a `thread_id` for its
  whole lifecycle, created via `mesh_thread` at assignment time?
- Does a `WorkTask` state transition (assigned → in-progress → done)
  get announced as a `mesh_send` on the task's thread, or does
  `TickEngine` poll `GardenHub` state directly and only use
  `colony-mesh` for cross-agent chatter, not task-state itself?
- Recommended default, to keep the slice small: **`WorkTask` state
  lives in `GardenHub` as it does today (no change there); a
  `colony-mesh` thread is created per `WorkTask` only for the
  identity's own chatter/evidence-posting, using `{agent}@soffio` as
  the instance id.** This avoids making `colony-mesh` a second source
  of truth for task state in the slice — defer that harder question
  (does colony-mesh become the state store?) to after the slice proves
  the simpler shape works.

thoor: this default is proposed, not decided — flag if you want the
harder shape (mesh as task-state source of truth) considered now
instead of deferred.

## D1 — Self-responsibility gate v0 (design decision, blocking Track C)

Not the full Venice loop machinery (streak loops, holdout tests,
hill-climbing) — the minimal evidence contract for one `WorkTask`
completion claim in this slice:

- Proposed v0: a `WorkTask` cannot transition to `done` without an
  attached evidence artifact — a test command's exit code + captured
  stdout/stderr, following the same "no log/SQL output = no claim"
  discipline seal-gate already enforces session-wide
  (`~/.claude/settings.json`'s SessionStart hook). Concretely: gate
  function takes `(workTaskId, evidence: {command, exitCode, output})`
  and only allows the `done` transition if `exitCode === 0` and
  `output` is non-empty.
- This is intentionally the smallest gate that's still real (rejects
  a bare "I'm done" claim), not a placeholder. Later pillar-2 work
  (Docker Sandboxes backend, streak loops, blank-arm holdout) extends
  this contract, doesn't replace it.

thoor: also proposed, not decided — flag if v0 should require more
than exit-code+output for the slice.

## Track A — Rialto minimal spawn (no blocking dependency, can start now)

Pure wiring against `agent-runtime`'s existing abstraction (see
`research/agent-runtime-and-colony-mesh-2026-08-12.md`) — no new
design needed:

- A1: Rialto constructs one `LoadedAgent` + `Provider` +
  `AgentSession` for a single identity from `data/colony.json`,
  reusing `src/agents/session.ts` as-is.
- A2: Smoke-test `modelOverride` actually switches the backing
  provider mid-flow for that identity (proves substrate-swap doesn't
  require new plumbing, per the earlier finding).

## Track B — colony-mesh participation for Soffio agents (blocked on D0)

- B1: Register `soffio` as a platform short-name alongside
  `anima`/`codex`/`claude-code`/`oc`; adopt `{agent}@soffio` as the
  instance-id convention.
- B2: Verify the existing `mesh_register_recipient` /
  `mesh_send` / `mesh_pull` / `mesh_ack` round trip works for a
  `soffio` instance id — this is already spec'd and tested
  infrastructure (`openspec/specs/colony-mesh-mcp-runtime-usage/spec.md`),
  so this is a verification task, not new code, unless the round trip
  reveals a gap for a 5th platform.
- B3: Implement the `WorkTask` → `mesh_thread` mapping per D0's
  decided contract.

## Track C — Self-responsibility gate v0 (blocked on D1)

- C1: Implement the minimal evidence-check function per D1's decided
  contract.
- C2: Wire the gate as a required step before a `WorkTask` can
  transition to `done` in this slice's flow (not yet a global
  `GardenHub` change — scoped to the slice's single `WorkTask` path
  first).

## Integration (blocked on A + B + C all landing)

- I1: Wire the full chain: Rialto spawns the identity (A) → identity
  registers on `colony-mesh` (B) → identity works its one `WorkTask` →
  gate checks evidence before allowing `done` (C) → completion posted
  to the `WorkTask`'s mesh thread.

## Verify (blocked on I1)

- V1: Run the slice for real, once, end-to-end. Capture real output
  (not illustrative) — same standard as debug-skill's README
  examples this session already set.
- V2: Negative test — attempt to mark the `WorkTask` done with no
  evidence or a failing exit code, confirm the gate actually rejects
  it. This is the one test that proves the gate is real, not a
  formality — skipping it would repeat the exact mistake Venice loop
  engineering exists to prevent.

## What this plan deliberately leaves open

Per thoor's "cần discuss thêm": pillar 2's full shape beyond v0
(Docker Sandboxes backend, streak loops, holdout tests), pillar 4's
background-listener mechanism and whether it lives in Rialto or
`colony-mesh` itself, and MVP ordering *beyond* this one slice (which
pillar/feature comes after the slice proves out). D0 and D1 above are
the two decisions this plan does need thoor's sign-off on before
Track B / Track C start — Track A has no such gate and can begin
immediately.

## Parallelization summary

Can start **right now, in parallel**: D0 (design decision), D1
(design decision), Track A (no dependency on either).

Can start **once D0 lands**: Track B.
Can start **once D1 lands**: Track C.
Track A, B, C can all run **in parallel with each other** once their
respective blockers clear — none of A/B/C depends on the others.

Integration and Verify are strictly sequential after all three tracks
land.
