# Intent: track-a-rialto-spawn

## Raw Request

"Vậy em plan cụ thể hết đi, và xem mục nào có thể làm song song."
(thoor, 2026-08-13, following Coda's proposed vertical-slice MVP:
1 identity + 1 WorkTask + provider/model-override + one colony-mesh
message + evidence-gated completion.) Track A is the first of the two
parallel-eligible implementation tracks from that plan — see
[`../../../plan-mvp-slice.md`](../../../plan-mvp-slice.md).

## Problem

Soffio is one harness process paired with one identity (see
[`../../../intent.md`](../../../intent.md)). Before that identity can
do any work, Soffio needs a way to obtain its substrate (memory,
config, continuity state) and actually run a turn against it. Without
this, nothing else in Soffio — the gate, the mesh participation, the
background listener — has an agent to operate on.

## Desired Outcome

Given a named identity that already has its own runtime running
(`coda-runtime` for the MVP slice), Rialto can join that runtime,
obtain the identity's substrate, and run one real turn using `pi`'s
own agent loop — with the underlying model swappable per run via
`pi`'s existing provider mechanism, no new abstraction required.

## Users / Actors

- Rialto (the component inside a Soffio process that performs this
  join-and-run path).
- The identity being run (Coda, for the MVP slice).
- `coda-runtime` (the identity's own runtime process, joined but not
  modified).

## Current Context

- `coda-runtime` (`packages/coda-runtime/` in the `anima` repo) is the
  only identity runtime that exists today. It exposes `POST /v1/join`
  and `POST /v1/continuity/remember` over HTTP
  (`packages/coda-runtime/src/http.ts`) — nothing resembling
  "execute a tool" or "run a turn."
- `pi` (the harness Soffio forked from) already ships a complete agent
  loop: `packages/agent/src/agent.ts` + `agent-loop.ts`. This is
  Soffio's own inherited code, used unmodified.
- An earlier draft of this design wrongly proposed a new
  `SoffioRuntime` class and wrongly proposed reusing ANIMA's own
  `AgentSession` — both corrected 2026-08-13, see
  [`../../../intent.md`](../../../intent.md)'s Revision History and
  this change's own `design.md`.

## Proposed Direction

Rialto's join-client speaks `coda-runtime`'s existing join protocol to
obtain the identity's substrate, then hands that substrate to `pi`'s
own agent loop as its system/context input. See `design.md` for the
full architecture and the two open questions (does the MVP slice
target Coda specifically; what shape `CodaJoinResponse`'s substrate
payload needs to be in before it can seed `pi`'s loop).

## Scope

- Joining one identity's runtime (`coda-runtime`) to obtain substrate.
- Running one turn of that identity via `pi`'s own agent loop.
- Proving model-override works mid-flow via `pi`'s own mechanism.

## Non-Goals

- Not building a new runtime class for Soffio (`SoffioRuntime` was
  proposed and explicitly rejected — see Revision History in
  `../../../intent.md`).
- Not reusing or importing ANIMA's `AgentSession` — unrelated code,
  ANIMA's own loop for ANIMA's own agents.
- Not building `aria-runtime`/`rina-runtime`/other identity runtimes —
  out of scope, upstream work if/when other identities need this.
- Not wiring colony-mesh participation (Track B) or the
  self-responsibility gate (Track C) — separate slices.

## Constraints

- Must reuse `pi`'s agent loop unmodified — no forking or duplicating
  its tool-execution/chat logic.
- Must not modify `coda-runtime` — Rialto is a client of its existing
  protocol, not a contributor to its codebase (that's ANIMA-repo work,
  out of scope here).

## Success Criteria

- A named identity (Coda) can be joined via `coda-runtime`'s real
  running process, not a mock.
- `pi`'s agent loop runs a real turn for that identity using the
  obtained substrate.
- A model-override smoke test shows the provider actually swaps
  mid-flow, using `pi`'s own mechanism.

## Risks

- `coda-runtime` reachability from wherever Rialto runs is unconfirmed
  (same host assumed for the MVP slice; Tailscale/remote reachability
  unverified) — see design.md's dependency note.
- The exact shape needed to feed `CodaJoinResponse`'s substrate into
  `pi`'s loop is not yet verified against `pi`'s actual loop code.

## Ambiguities

### Blocking

- Does the MVP slice target Coda specifically, or does thoor want a
  different identity's runtime built first? (Open question, design.md.)

### Non-Blocking

- Whether an existing `/v1/join` caller's request shape has
  undocumented conventions worth matching — doesn't block starting.

## Assumptions

- `coda-runtime` is already running and reachable for the MVP slice
  (locally, same machine as Rialto).
- `pi`'s agent loop can be fed an arbitrary system/context payload
  without requiring `pi`'s own provider-config bootstrapping to have
  run first — needs verification, not yet confirmed.

## Spec Seeds

- `soffio` requires `--agent <name>`; without it, prints help and
  exits before any join/run work (thoor, 2026-08-13, grounded in
  `pi`'s existing `parseArgs`/`printHelp` in
  `packages/coding-agent/src/cli/args.ts`).
- Rialto joins a named identity's runtime and obtains its substrate.
- `pi`'s agent loop runs a real turn using that substrate.
- Model-override switches the provider mid-flow via `pi`'s own
  mechanism.
- An unknown identity name is rejected before any join attempt.

## Intent Approval

Status: DRAFT

Approved by:
Date:
