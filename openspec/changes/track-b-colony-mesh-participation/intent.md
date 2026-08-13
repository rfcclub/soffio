# Intent: track-b-colony-mesh-participation

## Raw Request

"Vậy em plan cụ thể hết đi, và xem mục nào có thể làm song song."
(thoor, 2026-08-13, following Coda's proposed vertical-slice MVP.)
Track B is the second of the two parallel-eligible implementation
tracks from that plan — see
[`../../../plan-mvp-slice.md`](../../../plan-mvp-slice.md).

## Problem

The identity a Soffio process runs needs a way to talk to its peers —
other identities, other Soffio-run agents, Garden Hub's own agents —
without Soffio inventing a second message bus next to the one ANIMA's
colony already uses.

## Desired Outcome

An identity running under Soffio can register on `colony-mesh` (the
existing cross-runtime message bus) under a `soffio` platform
short-name, and use the existing `mesh_send`/`mesh_pull`/`mesh_ack`/
`mesh_thread` tools exactly as `anima`/`codex`/`claude-code`/`oc`
runtimes already do.

## Users / Actors

- The identity running under Soffio (sender/receiver of mesh messages).
- Other colony-mesh participants (Garden Hub's own agents, other
  runtimes) — Track B doesn't change their side at all.

## Current Context

- `colony-mesh` already exists and works: `src/mesh/anima-adapter.ts`
  (`createMeshTools`), `src/tools/a2a.ts`, `src/memory/a2a-store.ts`,
  10 MCP tools per
  `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md` (in the
  `anima` repo).
- `platform`/`to_platform` fields are free strings at the store layer
  — confirmed no ANIMA-side enum needs extending for a new platform
  value (`src/mesh/anima-adapter.ts` hardcodes its own `platform:
  "anima"` per-adapter).
- `colony-mesh` already covers 2 of AgentRadio's 3 primitives
  (`mesh_send` ≈ `send_message`, `mesh_thread` ≈ `create_thread`) —
  see [`../../../intent.md`](../../../intent.md) pillar 4.

## Proposed Direction

A `soffio` platform short-name, `{agent}@soffio` instance ids, a thin
Rialto-side mesh client wrapping the existing MCP tools (no new
server, no new store). Per `plan-mvp-slice.md`'s D0 decision, a
`WorkTask` gets a `colony-mesh` thread for chatter/evidence-posting
only — `WorkTask` state itself stays in Garden Hub. See `design.md`
for the open question on where `platform: "soffio"` registration
actually happens (likely `coda-runtime`'s own launch config, an
ANIMA-repo change).

## Scope

- Registering `soffio` as a valid platform for mesh participation.
- Send/pull/ack/thread round trip for a `soffio` instance id.
- One `WorkTask` thread for chatter, not state.

## Non-Goals

- Not building a new message bus or protocol — `colony-mesh` as-is.
- Not making `colony-mesh`'s `mesh_pull` a background listener yet —
  that's Track/pillar 4 (AgentRadio background listener), separate
  and still pending discussion.
- Not deciding whether `WorkTask` state should ever live in the mesh
  — explicitly deferred per D0's simpler default.

## Constraints

- Must not modify `A2AStore` or the delivery queue — client only.
- Must not require a change to any existing runtime's mesh adapter.

## Success Criteria

- A `soffio`-platform instance can register, send, and receive a
  message through the real running `colony-mesh` MCP server (not
  mocked).
- Cross-platform send (soffio → anima) delivers correctly with
  `from_platform` set to `"soffio"`.

## Risks

- Where `platform: "soffio"` registration should live is unresolved
  (Rialto-side vs. `coda-runtime`'s own launch config) — see design.md
  open question 3.

## Ambiguities

### Blocking

- Where does `platform: "soffio"` registration actually happen —
  needs thoor's input before task-level planning (design.md open
  question 3).

### Non-Blocking

- Whether `colony-mesh-mcp-runtime-usage/spec.md` needs a documented
  `soffio` section — doesn't block starting the client-side work.

## Assumptions

- The running `colony-mesh` MCP server is reachable from wherever
  Rialto runs, same as it is for existing runtimes.

## Spec Seeds

- `soffio` is a valid colony-mesh platform without upstream schema
  changes.
- Send/pull/ack round trip works for a `soffio` instance id.
- A `WorkTask` gets a mesh thread for chatter without duplicating task
  state into the mesh.

## Intent Approval

Status: DRAFT

Approved by:
Date:
