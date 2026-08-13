## Why

Soffio's original scoping assumed a new substrate-aware spawn API was
needed. Research (2026-08-12,
`../../../research/agent-runtime-and-colony-mesh-2026-08-12.md`)
found ANIMA's `agent-runtime` already separates identity from
provider call: `src/providers/interface.ts`'s `Provider` abstraction
and `src/agents/session.ts`'s `AgentSession` already run any
`LoadedAgent` against any `Provider` with a per-run `modelOverride`.
Rialto's spawn path should wire into this existing machinery, not
duplicate it. This change specs that wiring as its own reviewable
slice, decoupled from Track B (colony-mesh participation) and Track C
(self-responsibility gate), per
`../../../plan-mvp-slice.md`.

## What Changes

- Rialto gains a spawn path that constructs one `LoadedAgent` (via the
  existing `src/agents/loader.ts` loading logic, pointed at a
  colony identity's room) and one `Provider` (via
  `src/providers/registry.ts`'s `ProviderRegistry.resolve()`), then
  runs an `AgentSession` against them.
- A smoke-test proves `modelOverride` actually switches the backing
  provider mid-flow for a live identity, without any new abstraction
  — confirming pillar 1 of `../../../intent.md` needs no new API.
- No changes to `agent-runtime` itself. This is Rialto calling
  existing code, not extending it.

## Capabilities

### New Capabilities
- `rialto-spawn`: Rialto can spawn one colony identity as a live
  `AgentSession` backed by any registered `Provider`, with the
  provider/model swappable per run via `modelOverride`.

### Modified Capabilities
(none — `agent-runtime` is used as-is)

## Impact

- Affected code (Soffio side, new): Rialto's spawn module (path TBD
  in design.md).
- Affected code (ANIMA side, read-only dependency, not modified):
  `src/agents/loader.ts` (`LoadedAgent`, `AgentLoaderOptions`),
  `src/agents/session.ts` (`AgentSession`, `RunAgentSessionOptions`),
  `src/providers/interface.ts` (`Provider`, `ProviderRequest`),
  `src/providers/registry.ts` (`ProviderRegistry`).
- Dependency: this change assumes Soffio can import/require
  `agent-runtime`'s TypeScript modules directly (in-process) —
  confirming that assumption (vs. subprocess/RPC) is this change's
  first design decision, see design.md.
