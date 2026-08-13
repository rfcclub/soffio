## Architecture

**Track B rides on Track A's `SoffioRuntime`, not a separate mesh
client.** `packages/coda-runtime/src/runtime.ts`'s `CodaRuntimeOptions`
already has `mesh?: MeshRegistrationPort` with a single method,
`registerRecipient({instance_id, agent_name, platform, active})`.
`SoffioRuntime` (Track A) should accept the same option. Track B's
scope narrows to:

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

- **`SoffioMeshAdapter`** (new) — implements `MeshRegistrationPort`
  plus thin wrappers for `mesh_send`/`mesh_pull`/`mesh_ack`/
  `mesh_thread`, calling the existing `colony-mesh` MCP server exactly
  as `anima-adapter.ts`'s `createMeshTools()` does for ANIMA's own
  runtime — same tool surface, different `platform` value.
- **No new server, no new store.** `A2AStore`
  (`src/memory/a2a-store.ts`) and the MCP tool surface
  (`src/tools/a2a.ts`) are used as-is.

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
- Track A's `SoffioRuntime` for `MeshRegistrationPort` wiring — Track
  B can be developed and tested against a bare `SoffioMeshAdapter`
  independent of Track A landing first (per `plan-mvp-slice.md`'s
  dependency graph: A and B are parallel, only meet at Integration),
  but final wiring into `SoffioRuntime`'s constructor happens at
  Integration.

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
