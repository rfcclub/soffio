# agent-runtime and colony-mesh — 2026-08-12

thoor's correction to the previous session's Soffio pillar scoping:
pillar 1 (substrate-aware spawn) is largely already solved by
ANIMA's existing `agent-runtime`, and pillar 3 (Rialto's wire
protocol) should check `colony-mesh` for an existing A2A system
before designing anything new. Both turned out true — recorded here
with file citations, same discipline as the other research docs in
this directory.

## agent-runtime already has substrate-aware identity/provider separation

`src/providers/interface.ts` defines the `Provider` abstraction every
substrate implements identically:

- `ProviderName = "claude" | "gemini" | "qwen" | "codex" | "router" |
  (string & {})` — open-ended, not a closed enum, so adding a new
  substrate (Kimi, DeepSeek) doesn't require a type change elsewhere.
- `ProviderRequest` carries `agent: string` (identity, not substrate),
  `model?: string` (override), `mode: PermissionMode`, `effort?:
  'fast'|'normal'|'deep'`, `thinking?: {budget}`, and — notably —
  `staticHash`/`cachePrefix` fields already wired for prompt-cache
  breakpoints (Anthropic/DeepSeek/MiniMax `cache_control`-style). This
  is the same "stable prefix first" prompt-layering concern flagged as
  an open question in the earlier feature-adoption research, already
  solved here at the interface level.
- `Provider.query(req)` returns either a streaming
  `AsyncGenerator<ProviderChunk>` or a `Promise<ProviderResponse>` —
  one call shape regardless of which substrate backs it.

`src/agents/session.ts` (`AgentSession`, `RunAgentSessionOptions`,
`AgentSessionOptions`) runs one `LoadedAgent` (an identity — name,
config, cartridge, continuity DB path) against any injected
`Provider`, with `modelOverride` as a per-run parameter independent of
the identity's own state. The identity's continuity
(`continuityDbPath`, defaulting to `~/.anima/agents/<agent>/continuity.db`),
memory store, and cartridge are all keyed by agent name, not by which
provider/model answered the last turn — i.e. the separation Soffio's
pillar 1 wanted ("identity is not a provider call") is the existing
data model, not a new one to build.

**What this changes for Soffio**: pillar 1 stops being "design a
substrate-aware spawn API from scratch, compare against `pi`'s
`packages/ai`" and becomes "wire Rialto's bulk-spawn path into this
existing `Provider`/`AgentSession` layer." The open question that
remains is the *wiring shape* — does Rialto call `agent-runtime` as an
in-process library, shell out to it, or go through `colony-mesh` — not
the underlying abstraction, which already exists and is exercised in
production by ANIMA today.

## colony-mesh already has a working, cross-runtime A2A system

Not a design doc — a running system with an MCP surface, a spec, and
migration tooling:

- **Store**: `src/memory/a2a-store.ts` (`A2AStore`) — `.send()`,
  `.readInbox()`, `.close()`.
- **Tool layer**: `src/tools/a2a.ts` (`createA2ATools`,
  `A2AToolContext`, `handleSendMessage`, `handleReadInbox`).
- **Mesh adapter**: `src/mesh/anima-adapter.ts` (`createMeshTools`) —
  exposes the mesh as MCP tools.
- **Spec** (`openspec/specs/colony-mesh-mcp-runtime-usage/spec.md`):
  the MCP server exposes exactly 10 tools — `mesh_send`, `mesh_peek`,
  `mesh_pull`, `mesh_ack`, `mesh_status`, `mesh_thread`,
  `mesh_subscribe`, `mesh_unsubscribe`, `mesh_register_recipient`,
  `mesh_unregister_recipient` — and this is a tested requirement
  (`test/a2a-tools.test.ts`, `test/a2a-store.test.ts`), not aspirational.
  The spec also requires distinct `MESH_INSTANCE_ID` per runtime (no
  shared `cli-aria` id) and documents Codex, ClaudeCode, Qwen, Gemini,
  Antigravity as already-configured runtimes — **cross-runtime already
  works today**, which is exactly the property Rialto needed to build.
- **Docs** (`docs-site/colony/mesh-bus.md`): claim/ack delivery
  semantics (`mesh_pull` claims with a `claim_token`, `mesh_ack`
  finalizes; failed ack leaves the message re-claimable, so handlers
  must be idempotent), pub/sub topics (`mesh_subscribe`/`mesh_send`
  with a `topic`), threads (`mesh_thread`, first message with a
  `thread_id` creates it, read-only history), cross-runtime routing by
  `instance_id` = `{agent_name}@{platform_short}` (e.g.
  `aria@anima`, `coda@codex`, `lyra@claude-code`, `2b@oc`).

**Mapping onto AgentRadio's three primitives** (see
[`ai-agent-digest-2026-08-10.md`](./ai-agent-digest-2026-08-10.md) for
the original AgentRadio research):

| AgentRadio primitive | colony-mesh equivalent | Gap |
|---|---|---|
| `create_thread(name, participants)` | `mesh_thread` (thread created implicitly by first `thread_id`-tagged message) | None significant — same shape |
| `send_message(thread, content, mentions)` (non-blocking) | `mesh_send` (`bestEffort: true` = fire-and-forget, matches non-blocking) | None significant |
| `wait_for_mention(timeout)` run as a **background task** | `mesh_pull` — but only as a **foreground poll** a session actively calls | **This is the actual gap.** colony-mesh has claim-based pull, not a background listener that surfaces mentions between an agent's own work steps. |

**What this changes for Soffio**: pillar 3 (GardenHub/Rialto wiring)
stops being "design a protocol inspired by catcode's
`protocol.schema.json`" and becomes "make bulk-spawned Soffio agents
first-class colony-mesh participants" — likely a `soffio` platform
short-name, `{agent}@soffio` instance ids, same as `anima`/`codex`/
`claude-code`/`oc` today. Pillar 4 (AgentRadio pattern) narrows from
"build three primitives" to "build the one missing piece": a
background listener that turns `mesh_pull` from an actively-called
foreground poll into something that surfaces mentions at an agent's
next step boundary without it stopping work to listen — the same
distinction AgentRadio's paper isolated and measured.

Whether that background-listener piece belongs inside Rialto
specifically, or as a `colony-mesh` enhancement any runtime could use
(ANIMA, Codex, Claude Code, OpenClaw — not just Soffio-spawned
agents), is explicitly not decided — thoor, 2026-08-12: pillar 4 (and
pillar 2, and MVP ordering) still need discussion before any of this
gets built.

## What's now open for Soffio's design phase, narrowed

Before this research: 4 pillars, all needing a design pass.

After: pillar 1 is mostly an integration task against existing
`agent-runtime`; pillar 3's protocol question is "use colony-mesh,"
not "invent a protocol"; pillar 4 narrows to one missing primitive
layered onto colony-mesh rather than three built from scratch. Pillar
2 (self-responsibility gate specifics) and the MVP slice ordering
remain fully open, plus the exact wiring shape for pillars 1 and 3 —
see Open Questions in [`../intent.md`](../intent.md).
