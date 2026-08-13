## ADDED Requirements

### Requirement: soffio is a valid colony-mesh platform without upstream changes
`colony-mesh`'s `platform`/`to_platform` fields are free strings at
the store layer (confirmed: `src/mesh/anima-adapter.ts` hardcodes
`platform: "anima"` in its own outgoing messages and casts
`to_platform` as `string`, with no enum validation in
`src/memory/a2a-store.ts`). Soffio SHALL register as `platform:
"soffio"` via its own thin adapter, the same pattern `anima-adapter.ts`
uses for `"anima"`, requiring no change to `colony-mesh`'s store or
existing runtime adapters.

#### Scenario: Soffio-spawned identity registers with a soffio instance id
- **WHEN** a Rialto-spawned identity `"<name>"` calls
  `mesh_register_recipient`
- **THEN** it registers with `instance_id = "<name>@soffio"` and
  `platform: "soffio"`
- **THEN** no existing runtime's registered instance ids
  (`anima`/`codex`/`claude-code`/`oc`) are affected

### Requirement: Send/pull/ack round trip works for a soffio instance id
The existing `mesh_send`/`mesh_pull`/`mesh_ack` round trip SHALL work
identically for a `soffio` platform instance as it does for the four
already-configured runtimes — this is a verification requirement
against existing infrastructure, not new mesh functionality.

#### Scenario: Message sent to a soffio identity is received and acked
- **GIVEN** identity `"<name>"` is registered as `"<name>@soffio"`
- **WHEN** another mesh participant calls `mesh_send({ to_id: "<name>",
  to_platform: "soffio", content: "..." })`
- **THEN** `mesh_pull({ instance_id: "<name>@soffio" })` returns that
  message with a `delivery_id` and `claim_token`
- **THEN** `mesh_ack({ delivery_id, claim_token })` succeeds and the
  message is not re-deliverable on a subsequent `mesh_pull`

#### Scenario: A soffio identity sends a message to an existing-runtime agent
- **WHEN** a Rialto-spawned identity calls `mesh_send({ to_id: "aria",
  to_platform: "anima", content: "..." })`
- **THEN** the message is delivered to `aria@anima`'s inbox exactly as
  if the sender were any other existing runtime
- **THEN** `from_platform` on the delivered message reads `"soffio"`

### Requirement: A WorkTask gets a colony-mesh thread without duplicating task state into the mesh
Per `plan-mvp-slice.md`'s D0 decision, a `WorkTask`'s lifecycle state
SHALL remain in `GardenHub` (`src/garden/hub.ts`) as the single source
of truth. `colony-mesh` SHALL only host a thread for that `WorkTask`'s
chatter/evidence-posting, created via `mesh_thread` semantics
(first `thread_id`-tagged message creates it).

#### Scenario: WorkTask thread carries evidence, not state transitions
- **GIVEN** a `WorkTask` assigned to a Rialto-spawned identity
- **WHEN** the identity posts progress or evidence via
  `mesh_send({ thread_id: "<worktask-id>", ... })`
- **THEN** the message appears in `mesh_thread({ thread_id:
  "<worktask-id>" })`'s history
- **THEN** the `WorkTask`'s own status field in `GardenHub` is
  unchanged by that mesh message alone — a status transition requires
  the gate (Track C), not a mesh post
