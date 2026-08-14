## ADDED Requirements

### Requirement: a Rialto join yields an independently addressable mesh identity, with no separate registration step
`coda-runtime`'s `join()` already registers every joined instance on
`colony-mesh` automatically, deriving `instance_id` from
`slugifyInstanceId(agentId, surfaceId)` with no persisted "already
registered" state (`openspec/specs/coda-runtime-mesh-registration/spec.md`
in the `anima` repo — existing behavior, verified against live code,
not new mesh functionality this change implements). Rialto's join
call SHALL pass a `surfaceId` that yields a distinct, addressable mesh
identity.

#### Scenario: A Rialto join with a distinct surfaceId registers automatically
- **WHEN** Rialto joins identity `"<name>"` with `surfaceId:
  "soffio"` (or another agreed value)
- **THEN** the join's existing mesh-registration side effect fires,
  with no extra call from Rialto
- **THEN** the resulting `instance_id` is independently addressable
  from any other join of the same `agentId` with a different
  `surfaceId`

#### Scenario: A registration failure never fails the join
- **GIVEN** `mesh.registerRecipient()` throws during a Rialto join
- **WHEN** the join otherwise completes its continuity/lease flow
- **THEN** `join()` still returns a normal, complete
  `CodaJoinResponse` — Rialto does not need its own error-handling
  path for this failure mode (existing `coda-runtime` behavior)

### Requirement: Send/pull/ack round trip works for a Rialto-joined instance
The existing `mesh_send`/`mesh_pull`/`mesh_ack` round trip SHALL work
identically for a Rialto-joined instance as it does for any other
`colony-mesh` participant — a verification requirement against
existing infrastructure, not new mesh functionality.

#### Scenario: Message sent to a Rialto-joined identity is received and acked
- **GIVEN** identity `"<name>"` was joined with `surfaceId: "soffio"`
- **WHEN** another mesh participant sends a message to that instance's
  `instance_id`
- **THEN** `mesh_pull` (or the wakeup-triggered read, see below)
  returns that message with a `delivery_id` and `claim_token`
- **THEN** `mesh_ack({ delivery_id, claim_token })` succeeds and the
  message is not re-deliverable on a subsequent read

#### Scenario: A Rialto-joined identity sends a message to an existing-runtime agent
- **WHEN** a Rialto-joined identity sends a message to `aria@anima`
- **THEN** the message is delivered to `aria`'s inbox exactly as if
  the sender were any other existing runtime

### Requirement: a background listener surfaces mesh wakeups without polling
`colony-mesh` already has a real push/wakeup layer
(`openspec/archive/colony-mesh-realtime-pubsub-*/specs/mesh-realtime-pubsub/spec.md`
in the `anima` repo — publish persists to SQLite first, then emits a
wakeup to registered listeners; SQLite stays authoritative). Rialto
SHALL register a listener for the joined identity's `instance_id` and
surface wakeups to `pi`'s agent loop at its next step boundary,
without the agent blocking on a poll call.

#### Scenario: A wakeup surfaces without the agent polling
- **GIVEN** a listener is registered for the joined identity's
  `instance_id`
- **WHEN** another participant publishes a message to that
  `instance_id`
- **THEN** the listener receives a wakeup after the message is durably
  committed to `colony-mesh`'s store
- **THEN** the agent's ongoing turn is not blocked waiting for the
  wakeup — it surfaces at the next step boundary

#### Scenario: The wakeup is a hint, not the delivery itself
- **GIVEN** a wakeup has been received
- **WHEN** Rialto processes it
- **THEN** Rialto reads the actual delivery via `pull`/`peek` rather
  than treating the wakeup payload as the message content

### Requirement: A WorkTask gets a colony-mesh thread without duplicating task state into the mesh
Per `plan-mvp-slice.md`'s D0 decision, a `WorkTask`'s lifecycle state
SHALL remain in `GardenHub` (`src/garden/hub.ts`) as the single source
of truth. `colony-mesh` SHALL only host a thread for that `WorkTask`'s
chatter/evidence-posting, created via `mesh_thread` semantics
(first `thread_id`-tagged message creates it).

#### Scenario: WorkTask thread carries evidence, not state transitions
- **GIVEN** a `WorkTask` assigned to a Rialto-joined identity
- **WHEN** the identity posts progress or evidence via
  `mesh_send({ thread_id: "<worktask-id>", ... })`
- **THEN** the message appears in `mesh_thread({ thread_id:
  "<worktask-id>" })`'s history
- **THEN** the `WorkTask`'s own status field in `GardenHub` is
  unchanged by that mesh message alone — a status transition requires
  the gate (Track C), not a mesh post
