## Architecture

**Correction, thoor 2026-08-13 (see Track A's design.md): no
`SoffioRuntime` — Rialto is a join-client of each identity's own
`<name>-runtime`, not a runtime itself.** This changes where mesh
registration lives: `packages/coda-runtime/src/runtime.ts`'s
`CodaRuntimeOptions.mesh?: MeshRegistrationPort` is a hook inside
`coda-runtime` *itself* — mesh registration for a Coda-spawned-via-
Rialto identity is `coda-runtime`'s own concern when it's configured
with `platform: "soffio"`, not something Rialto's join-client
performs on the identity's behalf from outside. Track B's scope
narrows to:

1. Constructing a `MeshRegistrationPort` implementation that calls the
   real `colony-mesh` MCP tools (`mesh_register_recipient`, and the
   send/pull/ack/thread tools beyond registration).
2. Passing `platform: "soffio"` and instance ids of the shape
   `"<name>@soffio"` — confirmed free-form at the store layer (no
   ANIMA-side enum to extend; `src/mesh/anima-adapter.ts` hardcodes
   its own `platform: "anima"` per-adapter, `to_platform` is cast as
   `string` with no validation).
3. A `WorkTaskThread` helper: creates/uses a `mesh_thread` per
   `WorkTask` id, per the spec's D0 decision that `WorkTask` state
   stays in `GardenHub` and the mesh thread only carries chatter/
   evidence.

## Components

- **`SoffioMeshAdapter`** (new, scope now uncertain post-correction —
  see below) — thin wrappers for `mesh_send`/`mesh_pull`/`mesh_ack`/
  `mesh_thread`, calling the existing `colony-mesh` MCP server exactly
  as `anima-adapter.ts`'s `createMeshTools()` does for ANIMA's own
  runtime. This part still holds regardless of the Track A correction
  — Rialto (or whatever process ends up owning WorkTask-thread
  chatter) needs *some* client for `mesh_send`/`mesh_pull`/
  `mesh_thread`, independent of registration.
- **No new server, no new store.** `A2AStore`
  (`src/memory/a2a-store.ts`) and the MCP tool surface
  (`src/tools/a2a.ts`) are used as-is. Still true.
- **`MeshRegistrationPort` wiring — open, not designed yet.** The
  original plan routed this through a `SoffioRuntime` constructor
  option that no longer exists (Track A's correction). `coda-runtime`
  registers its OWN `platform`/`mesh` config
  (`CodaRuntimeOptions.mesh`, `CodaRuntimeOptions.platform` per
  `runtime.ts`) — meaning for a Coda identity joined through Rialto,
  whether it registers as `platform: "soffio"` on colony-mesh is
  determined by however `coda-runtime` itself is configured/launched,
  not by Rialto's join call. `JoinCodaInput`
  (`clientId, surfaceId, message, cartridgeOverride, hostContext`) has
  no field for a caller to signal "register me under a different
  platform." This is now a real open question, not a solved piece of
  the design — see Open Questions.

## Data Model

No schema changes. Reuses `colony-mesh`'s existing delivery/thread
model. The only new "data" is a naming convention:
`instance_id = "<agent>@soffio"`, `platform = "soffio"`.

## Test Strategy

| Scenario ID | Test File | Type |
|-------------|-----------|------|
| Soffio-spawned identity registers with a soffio instance id | `test/soffio-mesh-adapter.test.ts` | integration (real MCP server, real `mesh_register_recipient` call) |
| Message sent to a soffio identity is received and acked | `test/soffio-mesh-roundtrip.test.ts` | integration (real send→pull→ack against the running MCP server, matching the round-trip proof pattern `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md` already requires for the 4 existing runtimes) |
| A soffio identity sends a message to an existing-runtime agent | `test/soffio-mesh-roundtrip.test.ts` | integration (cross-platform: soffio → anima, asserts `from_platform` on the delivered message) |
| WorkTask thread carries evidence, not state transitions | `test/soffio-worktask-thread.test.ts` | integration (real `mesh_thread`/`mesh_send`, asserts `GardenHub`'s `WorkTask` status field is untouched by a mesh post alone) |

## Dependencies

- The running `colony-mesh` MCP server (already exists, already
  tested — this change is a client of it, not a builder of it).
- Track A's corrected architecture (join-client, no `SoffioRuntime`)
  means Track B's dependency on Track A is now about *whichever
  process's config* ends up owning `platform: "soffio"` registration
  — likely `coda-runtime`'s own launch config for the MVP slice's
  chosen identity, which may mean a Track B task lands in
  `packages/coda-runtime/` (ANIMA repo) rather than only in Soffio's
  repo. Confirm this before `loomkit plan`.

## Migration

Not applicable — net-new code, and confirmed no ANIMA-side schema/enum
needs extending for a new platform value.

## Open Questions (for thoor before `loomkit plan`)

1. Should `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md` (in
   the `anima` repo) gain a `soffio` runtime section like it has for
   Codex/ClaudeCode/Qwen/Gemini/Antigravity, or does Soffio being a
   pure MCP *client* mean no upstream spec change is warranted? Leaning
   toward "yes, add it" for consistency with that spec's own
   requirement ("MCP usage is documented for all CLI runtimes"), but
   not decided.
2. Confirm the D0 default from `plan-mvp-slice.md` (mesh thread for
   chatter only, `GardenHub` stays the state authority) is still what
   thoor wants before `mesh_thread`/`WorkTask` wiring gets planned at
   task level.
3. **New, from Track A's correction**: where does `platform: "soffio"`
   registration actually happen — inside `coda-runtime`'s own launch
   config (an ANIMA-repo change), or does the identity-runtime join
   protocol need a new field so a joining client like Rialto can
   request a platform override? Not designed yet, needs thoor's input
   before Track B can be planned at task level.
