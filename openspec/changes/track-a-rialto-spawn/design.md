## Architecture

**Correction, thoor 2026-08-13: no new runtime class. Each identity
already owns its own named runtime (`coda-runtime` today,
`aria-runtime`/`rina-runtime`/etc. as they get built) — that
one-runtime-per-identity convention is deliberate and Soffio does not
get to add a second, harness-scoped runtime abstraction
("`SoffioRuntime`") alongside it.** The previous version of this
design proposed exactly that and was wrong — recorded here instead of
silently deleted, per this repo's own discipline of keeping design
mistakes visible (see the debug-skill `RemoteAttachArgs` precedent).

**Corrected role: Rialto is a *join-client*, not a runtime.** For each
identity it spawns, Rialto connects to *that identity's own*
`<name>-runtime` process using the same join protocol
`CodaRuntime`/`startCodaRuntimeHttp()` already implements (`POST
/v1/join` with capability-token/lease admission, per
`packages/coda-runtime/src/http.ts` and `runtime.ts`) — the identical
protocol every runtime in the `<name>-runtime` family speaks, not a
Soffio-specific one. Rialto's actual new code is small: a client that
speaks this join protocol to N different identity runtimes (whichever
ones exist for the identities being spawned), plus whatever
orchestration (parallel spawn, tracking N sessions) is Rialto's own
job as the harness.

**What this means for identities that don't have a runtime yet**: if
the MVP slice's chosen identity doesn't have its own `<name>-runtime`
built yet, building that runtime is its own piece of work, upstream of
and outside Soffio's scope — not something Rialto's spawn path
absorbs. Practically, this means the MVP slice should target **Coda**
specifically, since `coda-runtime` is the only one that exists today —
flagged as an open question below rather than assumed.

**Where the chat/tool-execution loop actually runs — resolved via the
subagent analogy, thoor 2026-08-13.** `coda-runtime`'s HTTP surface
(`packages/coda-runtime/src/http.ts`) exposes exactly two operations —
`POST /v1/join` and `POST /v1/continuity/remember` (`recall` returns
409 "unimplemented") — and nothing resembling "execute this tool" or
"run a turn." That narrow surface is itself the answer to "how do you
manage server-side vs. client-side tool execution": **coda-runtime is
a continuity/substrate authority only — it loads substrate, saves
experience, is queried for information. It is not where the
chat/reasoning loop or client-side tools (Grep, ls, Bash, edits — the
"harness supplies tools" layer) run.**

That loop runs **client-side, inside Rialto**, using `agent-runtime`'s
existing `AgentSession`/`executeTool`/`buildToolDefinitions`
(`src/agents/session.ts`, `src/tools/executor.ts`) after Rialto has
joined and received the identity's substrate via `coda-runtime`.
Rialto calls back to `coda-runtime` only for continuity writes
(`remember`) or for any operation that genuinely needs centralized
state (a "server tool" — e.g. cross-identity memory queries) — not for
routine per-turn tool calls, which stay local exactly like a spawned
subagent's own Read/Bash/Edit calls stay inside that subagent's own
process and are never visible to (or routed through) whatever spawned
it.

**The `leaseId` returned by `join()` is the same kind of handle this
session's own subagent-spawn mechanism uses** (`agent_id`/`name` from
the `Agent` tool): an address for addressing *that* identity's session
going forward, not a result payload. Rialto owns its own local
registry mapping identity name → `{leaseId, capabilityToken,
lastActivity}` for however many identities it has joined —
`coda-runtime` has no visibility into and no need to know how many
other identities Rialto is concurrently managing, the same way a
harness's subagent registry is invisible to the subagent itself.

## Components

- **Rialto join-client** (new, Soffio-side, small) — a client that
  performs the same `POST /v1/join` → capability-token/lease flow
  `CodaRuntime`'s HTTP server expects, generalized only in the sense
  that it can target any `<name>-runtime`'s HTTP endpoint, not in the
  sense of reimplementing runtime logic. No new "runtime" concept on
  Soffio's side at all.
- **No new server-side component.** `coda-runtime` (and whichever
  `<name>-runtime` processes exist for other identities) already runs
  as its own process; Rialto does not stand up or own that process.
- **Modification to `ProviderRequest`/model resolution**: none, as
  before — `modelOverride` (Track A's second requirement) is entirely
  inside the identity's own runtime/`agent-runtime` call path, not
  something Rialto's join-client touches.
- **Rialto's local lease registry** (new, Soffio-side, in-process for
  the MVP slice) — maps identity name → `{leaseId, capabilityToken,
  lastActivity}` for every identity Rialto has joined. This is
  Rialto's own bookkeeping, invisible to `coda-runtime`, mirroring how
  a harness tracks its spawned subagents without the subagents
  knowing about each other.
- **Rialto's own `AgentSession` execution** (reuses
  `agent-runtime`'s `AgentSession`/`executeTool` as-is, in-process
  inside Rialto) — the actual chat/tool-execution loop, using the
  substrate received from `coda-runtime`'s join response. This is
  where `Provider`/`modelOverride` (Track A's second requirement)
  actually gets exercised.

## Data Model

None owned by Rialto. Lease/session state stays inside whichever
`<name>-runtime` process Rialto joins (`coda-runtime`'s own
`coda_runtime_sessions` SQLite table is that identity's business, not
something Rialto reads or duplicates).

## Test Strategy

| Scenario ID | Test File | Type |
|-------------|-----------|------|
| Spawn a single identity by name | `test/rialto-join-client.test.ts` | integration (real `coda-runtime` process running, real `POST /v1/join`, not mocked — matches this session's established "no illustrative output" standard) |
| Unknown identity name is rejected before any spawn work happens | `test/rialto-join-client.test.ts` | unit (rejected before any HTTP call is attempted — no runtime endpoint to guess at for an unknown identity) |
| Same identity, two different models, same session flow | `test/rialto-model-override.test.ts` | integration (real provider swap, asserts on real response usage metadata — exact mechanism depends on open question 2 below) |
| Invalid modelOverride ref surfaces a clear error | `test/rialto-model-override.test.ts` | unit |

## Dependencies

- `coda-runtime` (`packages/coda-runtime/`) as the one identity runtime
  that exists today — the join protocol Rialto's client speaks is
  defined by `startCodaRuntimeHttp()`
  (`packages/coda-runtime/src/http.ts`), read as a spec of the wire
  protocol, not as a library Rialto imports.
- Whichever `<name>-runtime` processes get built later for other
  identities — Rialto's join-client should not hardcode
  Coda-specific assumptions beyond what the MVP slice needs, but
  building those other runtimes is out of scope for Track A itself.

## Migration

Not applicable — net-new code in Soffio's own repo, no existing
Soffio behavior to migrate.

## Open Questions (for thoor before `loomkit plan`)

1. **Confirm MVP slice targets Coda specifically**, since
   `coda-runtime` is the only identity runtime that exists today —
   or does thoor want a different identity's runtime built first as
   prerequisite work (outside this change's scope either way)?
2. ~~Who calls `POST /v1/join` on `coda-runtime` today~~ — largely
   resolved above: the chat/tool loop runs client-side regardless of
   who else calls `/v1/join`, since `coda-runtime`'s own surface has
   no execute-tool endpoint to be "the" caller of. Still worth
   confirming whether an existing caller's request shape (beyond
   `JoinCodaInput`'s documented fields) has undocumented conventions
   Rialto should match, but this no longer blocks the architecture
   decision.
3. **New**: does `agent-runtime`'s `AgentSession`/tool-executor
   (`src/agents/session.ts`, `src/tools/executor.ts`) run correctly
   when instantiated from a separate process (Soffio/Rialto) against
   a substrate obtained via `coda-runtime`'s join response, or does it
   assume it's always running inside ANIMA's own `server.ts`/`cli.ts`
   process (e.g. relative paths, shared config, `~/.anima/` layout
   assumptions)? This needs one more read of `AgentSession`'s
   constructor/`RunAgentSessionOptions` against what a join response
   actually returns (`CodaJoinResponse`'s `projection`/
   `declaredSubstrateScopes` etc.) before task-level planning.
