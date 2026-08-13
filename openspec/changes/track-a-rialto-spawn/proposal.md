## Why

One Soffio process runs one identity (see `../../../intent.md`). To
run that identity, Soffio needs its substrate — which lives in that
identity's own runtime (`coda-runtime` today; one runtime per
identity is a deliberate ANIMA convention, not something Soffio
duplicates). Rialto's job for this slice is to *join* that runtime
(the same `POST /v1/join` protocol `coda-runtime` already implements)
and then run the actual turn using `pi`'s own agent loop
(`packages/agent/src/agent.ts` + `agent-loop.ts`), inherited whole
from the fork — not ANIMA's `AgentSession`, which is ANIMA's own loop
for ANIMA's own agents. This change specs that join-and-run path as
its own reviewable slice, decoupled from Track B (colony-mesh
participation) and Track C (self-responsibility gate), per
`../../../plan-mvp-slice.md`.

## What Changes

- Rialto gains a small join-client that performs `coda-runtime`'s
  `POST /v1/join` flow for one named identity and obtains that
  identity's substrate/capability token.
- The obtained substrate feeds `pi`'s own agent loop as its
  system/context input; the loop runs unmodified — this is Soffio's
  inherited code, not new code.
- A smoke-test proves `pi`'s own model-override mechanism switches the
  backing provider mid-flow for the joined identity, without any new
  abstraction — confirming pillar 1 of `../../../intent.md` needs no
  new spawn API, only a join-client.
- No changes to `coda-runtime` or `pi`'s agent loop themselves. This
  is Rialto calling existing code on both sides, not extending either.

## Capabilities

### New Capabilities
- `rialto-join`: Rialto can join a single identity's own runtime
  (`coda-runtime` for the MVP slice) to obtain its substrate, then run
  that identity's turn via `pi`'s own agent loop, with the
  provider/model swappable per run via `pi`'s existing mechanism.

### Modified Capabilities
(none — `coda-runtime` and `pi`'s agent loop are used as-is)

## Impact

- Affected code (Soffio side, new): Rialto's join-client module (path
  TBD in design.md).
- Affected code (Soffio side, reused unmodified): `pi`'s agent loop
  (`packages/agent/src/agent.ts`, `agent-loop.ts`).
- Affected code (ANIMA side, read-only dependency, not modified):
  `packages/coda-runtime/src/http.ts` (`startCodaRuntimeHttp`),
  `runtime.ts` (`CodaRuntime`, `JoinCodaInput`, `CodaJoinResponse`).
- Dependency: this change assumes `coda-runtime` is reachable over
  HTTP from wherever Rialto runs — confirming the exact shape of that
  reachability (same host? Tailscale? local only for the MVP slice?)
  is this change's first design decision, see design.md.
