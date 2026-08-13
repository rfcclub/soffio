## Why

Soffio's original scoping assumed Rialto needed its own wire protocol,
inspired by catcode's `protocol.schema.json`. Research (2026-08-12,
`../../../research/agent-runtime-and-colony-mesh-2026-08-12.md`) found
`colony-mesh` already gives ANIMA a working, tested, cross-runtime A2A
system — 10 MCP tools, spec'd requirements, existing runtimes
(`anima`, `codex`, `claude-code`, `oc`). Rialto should make
Soffio-spawned agents first-class `colony-mesh` participants instead
of inventing a second protocol. This also covers 2 of AgentRadio's 3
primitives (`mesh_thread` ≈ `create_thread`, `mesh_send` ≈
`send_message`) for free — see `../../../intent.md` pillar 4.

## What Changes

- `soffio` becomes a registered platform short-name alongside
  `anima`/`codex`/`claude-code`/`oc`, with `{agent}@soffio` as the
  instance-id convention for Soffio-spawned identities.
- Soffio-spawned agents can `mesh_register_recipient`, `mesh_send`,
  `mesh_pull`, `mesh_ack` through the existing `colony-mesh` MCP
  surface — verified for this 5th platform, not just assumed to work
  because the spec says it's runtime-agnostic.
- One `WorkTask` (per `../../../plan-mvp-slice.md`'s D0 decision) gets
  a `colony-mesh` thread created via `mesh_thread` at assignment time,
  used for the identity's own chatter/evidence-posting — `WorkTask`
  state itself stays in `GardenHub` (`src/garden/hub.ts`), not
  duplicated into the mesh.

## Capabilities

### New Capabilities
- `soffio-mesh-participation`: Soffio-spawned identities can send,
  receive, and thread messages through `colony-mesh` using the
  `{agent}@soffio` instance-id convention.

### Modified Capabilities
- `colony-mesh-mcp-runtime-usage`: the existing spec
  (`openspec/specs/colony-mesh-mcp-runtime-usage/spec.md` in the
  `anima` repo, not this repo) documents Codex/ClaudeCode/Qwen/Gemini/
  Antigravity as configured runtimes. Whether `soffio` needs an
  equivalent documented section there, or whether Soffio only needs
  to be a *client* of the existing MCP surface with no spec change
  upstream, is a design.md decision — see design.md's open question.

## Impact

- Affected code (Soffio side, new): Rialto's mesh-client module (path
  TBD in design.md) — calls the existing `colony-mesh` MCP tools, does
  not reimplement `A2AStore` or the delivery queue.
- Affected code (ANIMA side, read-only dependency, likely not
  modified): `src/mesh/anima-adapter.ts` (`createMeshTools`),
  `src/tools/a2a.ts`, `src/memory/a2a-store.ts`. If the platform
  short-name list is hardcoded somewhere and needs `soffio` added,
  that's a small ANIMA-side change to identify in design.md — not
  assumed yet.
- Depends on Track A only for having a spawned identity to attach an
  instance-id to; otherwise independent (per `plan-mvp-slice.md`'s
  dependency graph, Track A and Track B can be built in parallel and
  only need to meet at Integration).
