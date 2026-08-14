## Architecture

**Corrected, thoor 2026-08-14: mesh registration is already automatic
inside `coda-runtime`'s `join()` — not something Rialto or Track B
implements.** Live spec, read directly (not archived, current source
of truth): `openspec/specs/coda-runtime-mesh-registration/spec.md`
(`anima` repo).

- `join()` calls an injected `MeshRegistrationPort.registerRecipient()`
  on **every** successful join, unconditionally — no "first join"
  branch, no separate registration step for a caller to remember to
  perform.
- `instance_id` derives deterministically from `slugifyInstanceId(agentId,
  surfaceId)` — pure function, no persisted "have I registered before"
  state. **Not `platform`** — the earlier design (pre-2026-08-14) was
  wrong to route this through a `platform: "soffio"` field; the real
  mechanism keys on `surfaceId`, which `JoinCodaInput` already accepts
  as a caller-supplied field.
- Two joins with different `surfaceId` values (same `agentId`) get two
  independently addressable mesh recipients (spec's SC-010) — this is
  exactly what Rialto needs: join as `(agentId: "coda", surfaceId:
  "soffio")` or similar, and the resulting instance is automatically,
  distinctly addressable on `colony-mesh` — no extra code.
- Mesh registration failure never fails `join()` (SC-006) — Rialto
  doesn't need its own error-handling path for this either.

**Also corrected: `colony-mesh` already has a real push/wakeup layer,
not poll-only.** Archived spec (shipped, not aspirational):
`openspec/archive/colony-mesh-realtime-pubsub-*/specs/mesh-realtime-pubsub/spec.md`.
Publish persists to SQLite first, then emits a wakeup to any
registered listeners — multiple simultaneous listeners supported,
direct/broadcast/topic routing, listener expiry, wakeup-emission
failure never fails publish. SQLite stays authoritative; a wakeup is a
hint to read, not the delivery itself.

**What Track B actually still needs to do**, given both of the above
are already built:

1. Verify (not implement) that a Rialto-initiated join gets a working,
   independently-addressable mesh identity for free.
2. Verify send/pull/ack works for that identity against the real
   running mesh.
3. Register a listener for the existing wakeup layer and surface
   wakeups to `pi`'s agent loop at its next step boundary (this is
   pillar 4's real remaining work — a listener registration + a
   surfacing point, not a poll loop or a new push mechanism).
4. The `WorkTask`-thread-for-chatter piece (D0's decision, `GardenHub`
   stays the state authority) is unaffected by this correction — still
   real work, still Track B's own.

## Components

- **No new mesh adapter to build for registration** — `coda-runtime`'s
  `join()` already does this. Track B's join call (via Track A) just
  needs to pass a `surfaceId` that yields a sensible, distinct
  instance_id (e.g. `"soffio"`, or `"soffio-<host>"` if per-machine
  addressability matters later).
- **Thin mesh client** (still needed) — wrappers for `mesh_send`/
  `mesh_pull`/`mesh_ack`/`mesh_thread`, calling the existing
  `colony-mesh` MCP server, for whatever process ends up posting
  `WorkTask` chatter (Rialto, using the identity obtained via Track A).
- **Wakeup listener registration** (still needed, narrower than
  before) — register with the existing wakeup layer for the joined
  identity's `instance_id`; on wakeup, read the real delivery via
  `pull`/`peek` (per the wakeup spec's own non-authoritative-hint
  contract) and surface it to `pi`'s agent loop.
- **`WorkTaskThread` helper** (still needed) — creates/uses a
  `mesh_thread` per `WorkTask` id; `WorkTask` state itself stays in
  `GardenHub`, unchanged from the original D0 decision.
- **No new server, no new store.** `A2AStore`
  (`src/memory/a2a-store.ts`) and the MCP tool surface
  (`src/tools/a2a.ts`) are used as-is — still true, more so now.

## Data Model

No schema changes. Reuses `colony-mesh`'s existing delivery/thread/
wakeup model entirely. No new naming convention needed either —
`instance_id` is whatever `slugifyInstanceId(agentId, surfaceId)`
produces for the `surfaceId` Rialto's join call passes.

## Test Strategy

| Scenario ID | Test File | Type |
|-------------|-----------|------|
| A Rialto join with a distinct surfaceId yields a distinct, addressable mesh instance | `test/soffio-join-mesh-identity.test.ts` | integration (real `coda-runtime` join, real mesh registration, no mocks — verifies existing behavior, not new code) |
| Message sent to that instance is received and acked | `test/soffio-mesh-roundtrip.test.ts` | integration (real send→pull→ack against the running MCP server) |
| A wakeup for that instance surfaces without polling | `test/soffio-mesh-wakeup.test.ts` | integration (real publish, real listener registration, asserts the listener is notified before any manual `pull` call) |
| WorkTask thread carries evidence, not state transitions | `test/soffio-worktask-thread.test.ts` | integration (real `mesh_thread`/`mesh_send`, asserts `GardenHub`'s `WorkTask` status field is untouched by a mesh post alone) |

## Dependencies

- The running `colony-mesh` MCP server and its wakeup layer (both
  already exist, already tested — this change is a client of both,
  builds neither).
- Track A's join call, which must pass a `surfaceId` — the two changes
  now meet at exactly one shared decision (what `surfaceId` value to
  use), not at a registration mechanism to jointly design.

## Migration

Not applicable — no new code beyond thin clients/listeners, and
confirmed no ANIMA-side schema/enum change needed anywhere.

## Open Questions (for thoor before `loomkit plan`)

1. What `surfaceId` should Rialto's join call use — a fixed literal
   (`"soffio"`), something host-qualified (`"soffio-<hostname>"`), or
   something else? Affects whether multiple Soffio processes running
   the same identity on different machines get distinct or colliding
   mesh addresses.
2. Confirm the D0 default from `plan-mvp-slice.md` (mesh thread for
   chatter only, `GardenHub` stays the state authority) is still what
   thoor wants before `mesh_thread`/`WorkTask` wiring gets planned at
   task level.
3. Should `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md` (in
   the `anima` repo) gain a documented section for Soffio-originated
   instances, given they're really just `coda-runtime` joins with a
   distinct `surfaceId`, not a new "platform"? Lower-stakes now that
   there's no new registration mechanism to document — may not need
   its own section at all.
