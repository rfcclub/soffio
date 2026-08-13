## Architecture

**Core decision: Rialto follows ANIMA's existing "identity-runtime"
pattern, not a raw in-process import of `src/agents/`.**

Three options were on the table for how a Soffio-spawned identity
reaches `agent-runtime`:

1. **In-process import** — Soffio imports `src/agents/session.ts` /
   `src/providers/registry.ts` TS modules directly. Rejected: these
   modules are organized for ANIMA's own server
   (`src/server.ts`)/CLI (`src/cli.ts`), not published as a stable
   library API — any ANIMA-side refactor could silently break Soffio,
   and Soffio would need ANIMA's full dependency tree in-process.

2. **Ad-hoc subprocess/RPC** — build a new one-off CLI or HTTP
   endpoint in ANIMA just for Soffio to shell out to. Rejected before
   being seriously considered once option 3 was found: reinventing a
   pattern that already exists and is already proven for a real
   identity (Coda) would duplicate work and diverge from ANIMA's own
   architecture.

3. **Identity-runtime pattern (chosen)** — ANIMA already has a general
   package for exactly this problem: `packages/identity-runtime/`
   (`RemoteContinuityPort`, `ProjectionGrant`/`narrowProjectionGrant`,
   `ScopeMapper`, `RuntimeRegistry`, `IncarnationLease` via
   `continuity-core`) with `packages/coda-runtime/` as one concrete
   instantiation for the Coda identity. `CodaRuntime`
   (`packages/coda-runtime/src/runtime.ts`) is a **separate process**
   that owns its own SQLite-backed lease/mutation-outbox state
   (`CREATE TABLE coda_runtime_sessions`), joins ANIMA's continuity
   plane over HTTP (`packages/coda-runtime/src/http.ts`,
   `startCodaRuntimeHttp()` — `POST /v1/join`, `POST
   /v1/continuity/remember`) with capability-token/lease admission
   (`join()` calls `this.options.continuity.compile(...)`, returns
   `CodaJoinResponse` with `leaseId`, `capabilityToken`,
   `writeScopes`), and — notably — already has an optional
   `mesh?: MeshRegistrationPort` constructor option
   (`registerRecipient({instance_id, agent_name, platform, active})`)
   for Track B's exact concern. "Canonical prompt and memory authority
   remain behind `RemoteContinuityPort` in ANIMA" (verbatim from that
   file's own docstring) — i.e. this pattern is specifically designed
   so a peer process does NOT need ANIMA's internals in-process.

**Rialto's spawn path is therefore: build a `SoffioRuntime`, a peer to
`CodaRuntime`, instantiated once per spawned identity, using the same
`packages/identity-runtime/` primitives.** Not a new mechanism — the
second instantiation of an existing one.

## Components

- **`SoffioRuntime`** (new, Soffio-side) — mirrors
  `packages/coda-runtime/src/runtime.ts`'s shape: constructor takes
  `dbPath`, `continuity: RemoteContinuityPort`, `agentId`,
  `mesh?: MeshRegistrationPort`, `platform: "soffio"`. Exposes
  `join(input)` and whatever remember/recall surface the identity
  needs, following `CodaJoinResponse`'s shape (`leaseId`,
  `capabilityToken`, `writeScopes`).
- **`startSoffioRuntimeHttp`** (new, Soffio-side, optional) — mirrors
  `startCodaRuntimeHttp()` only if Rialto needs its spawned identities
  reachable over HTTP from outside its own process; if Rialto only
  ever calls `SoffioRuntime` in-process from its own spawn path, this
  wrapper may not be needed for the MVP slice — open question below.
- **Modification to `ProviderRequest`/model resolution**: none. The
  identity-runtime pattern doesn't change how `Provider`/
  `modelOverride` work (Track A's second requirement) — those stay
  exactly as `src/agents/session.ts` already implements them, reached
  through whatever `agent-runtime`-side hook `RemoteContinuityPort`'s
  concrete ANIMA-side implementation already uses to run a turn.

## Data Model

No new schema for the MVP slice beyond what `CodaRuntime` already
establishes as the pattern: a per-identity SQLite lease/session table
at `dbPath` (Soffio's own path, e.g.
`~/.soffio/agents/<name>/runtime.db`, mirroring ANIMA's
`~/.anima/agents/<agent>/continuity.db` convention noted in
`RunAgentSessionOptions.continuityDbPath`'s default).

## Test Strategy

| Scenario ID | Test File | Type |
|-------------|-----------|------|
| Spawn a single identity by name | `test/rialto-spawn.test.ts` | integration (real `SoffioRuntime` + real `agent-runtime` continuity port, not mocked — matches this session's established "no illustrative output" standard) |
| Unknown identity name is rejected before any spawn work happens | `test/rialto-spawn.test.ts` | unit |
| Same identity, two different models, same session flow | `test/rialto-spawn-model-override.test.ts` | integration (real provider swap, asserts on real `ProviderResponse` usage metadata) |
| Invalid modelOverride ref surfaces a clear error | `test/rialto-spawn-model-override.test.ts` | unit |

## Dependencies

- `packages/identity-runtime` and `packages/coda-runtime` (as a
  reference implementation to mirror, not a runtime dependency —
  `SoffioRuntime` is new code in Soffio's own repo following the same
  shape, since `coda-runtime` is Coda-specific by design, e.g. its
  `agentId ?? "coda"` default).
- Depends on `RemoteContinuityPort`'s ANIMA-side implementation
  exposing whatever hook actually runs an `AgentSession` turn — this
  is the one piece not yet traced in this design pass (the identity
  runtime docs describe the *admission/lease* side; the actual
  turn-execution call from a joined identity through to
  `AgentSession.run()` needs one more read of ANIMA's
  `RemoteContinuityPort` implementation before Track A's tasks.md can
  specify exact call sites).

## Migration

Not applicable — net-new code in Soffio's own repo, no existing
Soffio behavior to migrate.

## Open Questions (for thoor before `loomkit plan`)

1. Does Rialto need `startSoffioRuntimeHttp` (HTTP-reachable) for the
   MVP slice, or is in-process `SoffioRuntime` sufficient since Rialto
   itself is the only caller in the slice?
2. Confirm the `RemoteContinuityPort` → `AgentSession.run()` call path
   on ANIMA's side before task-level planning — this design pass found
   the admission/lease pattern but not yet the exact turn-execution
   hookup.
