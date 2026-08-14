## Why

Soffio's original scoping assumed Rialto needed its own wire protocol
(then, a "register as `platform: soffio`" step). Corrected 2026-08-14:
`coda-runtime`'s `join()` already registers every joined instance on
`colony-mesh` automatically (`openspec/specs/coda-runtime-mesh-registration/spec.md`
in the `anima` repo — live spec, existing behavior), keyed on
`(agentId, surfaceId)`, not a `platform` field. `colony-mesh` also
already has a real push/wakeup layer, not just poll
(`openspec/archive/colony-mesh-realtime-pubsub-*/`). Track B's actual
job shrinks to *verifying* both already work for a Rialto-initiated
join and *using* the wakeup layer for pillar 4 — not implementing
registration or a message bus.

## What Changes

- Rialto's join call (Track A) passes a `surfaceId` (e.g. `"soffio"`)
  that yields an independently addressable mesh `instance_id` — no new
  registration code, this already happens inside `coda-runtime`'s
  `join()`.
- A thin mesh client wraps `mesh_send`/`mesh_pull`/`mesh_ack`/
  `mesh_thread` for whatever process posts `WorkTask` chatter.
- A listener registers with the existing wakeup layer for the joined
  identity's `instance_id`, surfacing wakeups to `pi`'s agent loop at
  its next step boundary — this is pillar 4's real remaining work.
- One `WorkTask` (per `../../../plan-mvp-slice.md`'s D0 decision) gets
  a `colony-mesh` thread created via `mesh_thread` at assignment time,
  used for the identity's own chatter/evidence-posting — `WorkTask`
  state itself stays in `GardenHub` (`src/garden/hub.ts`), not
  duplicated into the mesh.

## Capabilities

### New Capabilities
- `soffio-mesh-participation`: a Rialto-joined identity can send,
  receive, thread, and get woken (without polling) on `colony-mesh`
  messages, using registration and push behavior `coda-runtime`/
  `colony-mesh` already provide.

### Modified Capabilities
(none — `colony-mesh` and `coda-runtime`'s registration/wakeup
behavior are used entirely as-is)

## Impact

- Affected code (Soffio side, new): a thin mesh client + wakeup
  listener registration (path TBD in design.md) — calls existing
  `colony-mesh` MCP tools and the existing wakeup layer, does not
  reimplement `A2AStore`, the delivery queue, or the wakeup mechanism.
- Affected code (ANIMA side, read-only dependency, not modified):
  `packages/coda-runtime/src/runtime.ts` (`join()`'s existing mesh
  registration), `src/mesh/anima-adapter.ts`, `src/tools/a2a.ts`,
  `src/memory/a2a-store.ts`, the wakeup/notify layer.
- Depends on Track A only for having a joined identity (and its
  `surfaceId` choice) to attach mesh activity to; otherwise
  independent (per `plan-mvp-slice.md`'s dependency graph, Track A and
  Track B can be built in parallel and only need to meet at
  Integration).
