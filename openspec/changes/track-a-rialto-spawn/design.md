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

**Where the chat/tool-execution loop actually runs — corrected,
thoor 2026-08-13: `AgentSession` is ANIMA's own, not Soffio's.**
The previous version of this design said the loop runs "using
`agent-runtime`'s existing `AgentSession`/`executeTool`" — wrong.
`src/agents/session.ts`'s `AgentSession` is ANIMA's own execution loop,
for ANIMA's own colony agents, running inside ANIMA's own process.
Soffio has zero reason to import or reuse it — Soffio is a fork of
`pi`, and `pi` already ships its own complete agent execution loop:
`packages/agent/src/agent.ts` + `packages/agent/src/agent-loop.ts`
(per this session's earlier harness-comparison research, ~18K + ~21K
lines — the actual tool-calling loop that makes `pi` a coding agent in
the first place). **Rialto uses `pi`'s own agent loop, unmodified, to
run the chat/tool-execution turn for a spawned identity.** No porting,
no cross-process compatibility question — it's already Soffio's own
code, inherited whole from the fork.

`coda-runtime`'s HTTP surface (`packages/coda-runtime/src/http.ts`)
exposes exactly two operations — `POST /v1/join` and `POST
/v1/continuity/remember` (`recall` returns 409 "unimplemented") — and
nothing resembling "execute this tool" or "run a turn." That narrow
surface confirms the split: **coda-runtime is a continuity/substrate
authority only — it loads substrate, saves experience. `pi`'s agent
loop is where the chat/reasoning loop and client-side tools (Grep, ls,
Bash, edits) run.** The two systems are bridged by Rialto, not merged:
Rialto joins `coda-runtime` to obtain the identity's substrate (feeds
it into `pi`'s agent loop as the system prompt / context), runs `pi`'s
own loop locally for the actual turn, then calls back to
`coda-runtime` only for continuity writes (`remember`) after the turn
— never for routine per-turn tool calls, which stay entirely inside
`pi`'s own loop, exactly like a spawned subagent's own Read/Bash/Edit
calls stay inside that subagent's own process and are never routed
through whatever spawned it.

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
- **Modification to `pi`'s own provider/model layer**: none —
  `modelOverride` (Track A's second requirement) is entirely `pi`'s
  own existing mechanism (`packages/ai`), not something Rialto's
  join-client or `coda-runtime` touches at all.
- **Rialto's local lease registry** (new, Soffio-side, in-process for
  the MVP slice) — maps identity name → `{leaseId, capabilityToken,
  lastActivity}` for every identity Rialto has joined. This is
  Rialto's own bookkeeping, invisible to `coda-runtime`, mirroring how
  a harness tracks its spawned subagents without the subagents
  knowing about each other.
- **`pi`'s own agent loop** (`packages/agent/src/agent.ts` +
  `agent-loop.ts`, used as-is, unmodified — inherited whole from the
  fork) — the actual chat/tool-execution loop, fed the substrate
  received from `coda-runtime`'s join response. This is where `pi`'s
  own provider/model-override mechanism (Track A's second requirement)
  actually gets exercised — not ANIMA's `Provider` interface.

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
3. ~~Does `agent-runtime`'s `AgentSession` run correctly outside
   ANIMA's own process~~ — moot, corrected above: Rialto never
   instantiates ANIMA's `AgentSession` at all. **New question in its
   place**: what shape does `CodaJoinResponse`'s `projection` need to
   be transformed into before it can seed `pi`'s own agent loop (system
   prompt? initial context messages? something else)? Needs one read
   of how `pi`'s `agent.ts`/`agent-loop.ts` accepts its initial
   system/context input before task-level planning.
