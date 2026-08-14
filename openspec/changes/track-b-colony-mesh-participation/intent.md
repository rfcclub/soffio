# Intent: track-b-colony-mesh-participation

## Raw Request

"Vậy em plan cụ thể hết đi, và xem mục nào có thể làm song song."
(thoor, 2026-08-13, following Coda's proposed vertical-slice MVP.)
Track B is the second of the two parallel-eligible implementation
tracks from that plan — see
[`../../../plan-mvp-slice.md`](../../../plan-mvp-slice.md). Corrected
2026-08-14 after thoor flagged that `colony-mesh` already has more
built than the original design assumed — see `design.md`'s
"Corrected" note and [`../../../intent.md`](../../../intent.md)'s
Revision History.

## Problem

The identity a Soffio process runs needs a way to talk to its peers —
other identities, other Soffio-run agents, Garden Hub's own agents —
without Soffio inventing a second message bus, second registration
mechanism, or second push/wakeup layer next to the ones ANIMA's
colony already uses and that `coda-runtime`'s `join()` already wires
up automatically.

## Desired Outcome

A Rialto-joined identity is automatically, independently addressable
on `colony-mesh` (no extra registration step — `coda-runtime`'s
`join()` already does this, keyed on `surfaceId`), can use the
existing `mesh_send`/`mesh_pull`/`mesh_ack`/`mesh_thread` tools, and
gets woken on new messages via the existing push/wakeup layer instead
of polling.

## Users / Actors

- The identity running under Soffio (sender/receiver of mesh messages,
  automatically registered via its own runtime's `join()`).
- Other colony-mesh participants (Garden Hub's own agents, other
  runtimes) — Track B doesn't change their side at all.

## Current Context

- `colony-mesh` already exists and works: `src/mesh/anima-adapter.ts`,
  `src/tools/a2a.ts`, `src/memory/a2a-store.ts`, 10 MCP tools per
  `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md`.
- **`coda-runtime`'s `join()` already registers every joined instance
  on `colony-mesh` automatically** — `openspec/specs/coda-runtime-mesh-registration/spec.md`
  (live spec, current behavior): `instance_id` derives from
  `slugifyInstanceId(agentId, surfaceId)`, no "first join" branch, no
  persisted registration state, registration failure never fails
  `join()`. This is *not* keyed on a `platform` field — the original
  Track B design assumed a `platform: "soffio"` registration step that
  doesn't match how the real mechanism works.
- **`colony-mesh` already has a real push/wakeup layer** —
  `openspec/archive/colony-mesh-realtime-pubsub-*/specs/mesh-realtime-pubsub/spec.md`:
  publish persists to SQLite first, then emits a wakeup to registered
  listeners; multiple simultaneous listeners, direct/broadcast/topic
  routing, listener expiry, wakeup-emission failure never fails
  publish. This already covers AgentRadio's `wait_for_mention`
  primitive — see [`../../../intent.md`](../../../intent.md) pillar 4.

## Proposed Direction

Rialto's join call (Track A) passes a `surfaceId` (e.g. `"soffio"`)
and gets an independently addressable mesh identity for free. A thin
mesh client wraps the existing send/pull/ack/thread tools. A listener
registers with the existing wakeup layer and surfaces wakeups to
`pi`'s agent loop at its next step boundary. Per `plan-mvp-slice.md`'s
D0 decision, a `WorkTask` gets a `colony-mesh` thread for chatter/
evidence-posting only — `WorkTask` state itself stays in Garden Hub.
See `design.md` for the remaining open question (what `surfaceId`
value to actually use).

## Scope

- Verifying a Rialto join yields an independently addressable mesh
  identity (not implementing registration — already exists).
- Send/pull/ack/thread round trip for that identity.
- A wakeup listener that surfaces mesh activity without polling.
- One `WorkTask` thread for chatter, not state.

## Non-Goals

- Not building a new message bus, registration mechanism, or
  push/wakeup layer — all three already exist and are used as-is.
- Not deciding whether `WorkTask` state should ever live in the mesh
  — explicitly deferred per D0's simpler default.
- Not modifying `coda-runtime`'s existing join/registration behavior.

## Constraints

- Must not modify `A2AStore`, the delivery queue, or the wakeup layer
  — client only.
- Must not require a change to any existing runtime's mesh adapter or
  to `coda-runtime`'s registration logic.

## Success Criteria

- A Rialto-joined identity is addressable on the real running
  `colony-mesh` MCP server with no extra registration code (not
  mocked).
- Send/receive/ack round trip works for that identity.
- A listener receives a wakeup (not a poll result) when a new message
  arrives for that identity.

## Risks

- What `surfaceId` value to standardize on is not yet decided — see
  design.md open question 1.

## Ambiguities

### Blocking

(none currently blocking — the registration/wakeup mechanism
questions that used to block this are resolved by using what already
exists)

### Non-Blocking

- What exact `surfaceId` convention to use (fixed literal vs.
  host-qualified) — doesn't block starting the verification work.
- Whether `colony-mesh-mcp-runtime-usage/spec.md` needs a documented
  section for Soffio-originated instances — lower-stakes now that
  there's no new registration mechanism to document.

## Assumptions

- The running `colony-mesh` MCP server and its wakeup layer are
  reachable from wherever Rialto runs, same as for existing runtimes.

## Spec Seeds

- A Rialto join with a distinct `surfaceId` registers automatically
  and is independently addressable — no separate registration call.
- A registration failure never fails the join.
- Send/pull/ack round trip works for a Rialto-joined instance.
- A background listener surfaces mesh wakeups without the agent
  polling or blocking.
- A `WorkTask` gets a mesh thread for chatter without duplicating task
  state into the mesh.

## Intent Approval

Status: DRAFT

Approved by:
Date:
