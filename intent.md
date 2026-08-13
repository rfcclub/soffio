# Soffio — Intent

## Name

**Soffio** (Italian, "breath" — as in *soffio di vita*, "breath of life").
`pi` (earendil-works) is the body being forked; the substrate (whichever
model backs an agent — Claude, Kimi, DeepSeek, Gemini, ...) is the soul.
Soffio is what connects the two and keeps it alive across substrate
changes.

**Rialto** — the orchestrator component *inside* Soffio (named after
Venice's original center, "Rivoalto" — the hub the city grew from).
Rialto is to Soffio what Garden Hub is to ANIMA's colony: the local
routing/visibility/coordination layer for whatever Soffio has spawned
— not a competing hub, a client/participant of the real one (Track B:
Rialto speaks `colony-mesh`, the same bus Garden Hub's own agents use).

**Correction, thoor 2026-08-13: "Venice Hub" is not a real system —
don't use that name.** `documents/GARDEN_PROPOSAL.md` deliberately
keeps two things separate that this doc's earlier drafts blurred
together under that invented name:
- **Garden Hub** (`src/garden/hub.ts`, `GardenHub`) — the colony
  control plane: which agents/runtime instances exist, tasks,
  blockers, events, memory proposals, routing. Real code, real API
  surface (`WorkTask`, `TickEngine`).
- **Venice loop engineering** (`castle-loop-engineering` spec, same
  doc) — the evidence/verification *discipline*, not a service with
  an API: mechanical rubric pass/fail, streaks, blank-arm holdout
  tests, human-gated hill-climbing. Soffio's self-responsibility gate
  (pillar 2 below) must satisfy this discipline's standard — Rialto
  doesn't "connect to" Venice loop engineering the way it connects to
  Garden Hub via colony-mesh; there's nothing to connect to. It's a
  bar the gate's evidence has to clear.

GARDEN_PROPOSAL.md's own words: "These are related, but they are not
the same system." Rialto's job touches both — participates in Garden
Hub via colony-mesh (Track B), and its gate (Track C) implements
Venice loop engineering's verification standard — but conflating them
into one "Venice Hub" was this doc's own error, now fixed.

## Problem

We already have general-purpose harnesses (Claude Code, Codex, Reasonix)
and reviewed four "coding agent CLI" forks/relatives (claw-code,
oh-my-pi, pi, catcode) to see what's out there. None of them are built
for what our colony specifically needs: they assume one agent per
session, one provider identity, and (at most) a generic
permission/sandbox model aimed at a single human developer's trust
boundary.

Our colony is different: 16 agents (`data/colony.json`), each with a
persistent identity that should survive a substrate/model swap, spawned
in bulk, coordinating through ANIMA's own event spine (GardenHub,
TickEngine), and expected to prove their own claims (Venice loop
engineering's evidence-gated verification) rather than self-report
"done."

We picked `pi` (earendil-works) as the base after a 4-way review:
strongest extension API (`packages/coding-agent/src/core/extensions/`,
`docs/extensions.md`), and — uniquely among the four — it reads Claude
Code's and Codex's skill directories directly (`~/.claude/skills`,
`~/.codex/skills`), so the colony isn't locked into one vendor's skill
format. Full review notes (claw-code, pi, catcode, oh-my-pi — what
each does well, ease-to-extend ratings, cited file paths) are kept in
[`research/harness-comparison-2026-08-09.md`](./research/harness-comparison-2026-08-09.md).

## Outcome

A fork of `pi` — Soffio — that adds four things `pi` doesn't have out
of the box:

1. **Substrate-aware spawn — largely already solved by `agent-runtime`,
   not a new build.** thoor's correction, 2026-08-12: "không cần thiết
   lắm vì ta có agent-runtime rồi." ANIMA's `src/providers/interface.ts`
   already defines the `Provider`/`ProviderRequest`/`ProviderResponse`
   abstraction every substrate (Claude, Codex, Qwen, Gemini, a router)
   implements identically, with per-request `model` override, `agent`
   name, `effort` tier, and prompt-cache fields (`staticHash`,
   `cachePrefix`) already built in. `src/agents/session.ts`
   (`AgentSession`, `RunAgentSessionOptions`) already runs one
   `LoadedAgent` identity against any `Provider` with a
   `modelOverride`, independent of the identity's own state
   (continuity DB, memory store, cartridge). This is the
   "identity separate from substrate call" mechanism Soffio needs —
   it exists today. Soffio's job for this pillar shrinks from
   "design a substrate-aware spawn API" to "wire Rialto's spawn path
   into `agent-runtime`'s existing `Provider`/`AgentSession` layer,"
   which is integration work, not new design. See
   [`research/agent-runtime-and-colony-mesh-2026-08-12.md`](./research/agent-runtime-and-colony-mesh-2026-08-12.md).

2. **Self-responsibility gate.** `pi` ships with no built-in
   permission/sandbox system by design (their choice, documented in
   `docs/containerization.md`) — fine for a single trusted developer,
   not fine for 16 autonomously-spawned agents. Soffio's gate is
   evidence-based, not a confirm-dialog: no "done" claim survives
   without the kind of proof Venice loop engineering already defines
   (CLI memory recall, mechanical verification, streak loops, blank-arm
   holdout tests, human-gated hill-climbing). The execution sandbox
   itself doesn't need to be hand-rolled: catcode's
   `MicrosandboxExecutionBackend` (microVM, see
   [`research/harness-comparison-2026-08-09.md`](./research/harness-comparison-2026-08-09.md))
   and **Docker Sandboxes** (docker.com/products/docker-sandboxes,
   see
   [`research/ai-agent-digest-2026-08-10.md`](./research/ai-agent-digest-2026-08-10.md))
   are both real, working microVM-isolation implementations of the
   same idea — thoor's call, 2026-08-11: Soffio should support Docker
   Sandboxes as a gate backend in the future. Deferred, not v1 — see
   Open Questions.

3. **GardenHub wiring — Rialto's wire protocol should be `colony-mesh`,
   not a new one.** thoor's steer, 2026-08-12: "Xem colony-mesh có
   A2A." It does, and it's real and running: `src/mesh/anima-adapter.ts`
   (`createMeshTools`), `src/tools/a2a.ts` (`createA2ATools`,
   `handleSendMessage`, `handleReadInbox`), `src/memory/a2a-store.ts`
   (`A2AStore`), exposed as 10 MCP tools per
   `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md`: `mesh_send`,
   `mesh_peek`, `mesh_pull`, `mesh_ack`, `mesh_status`, `mesh_thread`,
   `mesh_subscribe`, `mesh_unsubscribe`, `mesh_register_recipient`,
   `mesh_unregister_recipient` — cross-runtime already (Codex,
   ClaudeCode, Qwen, Gemini, Antigravity, per that spec's own
   requirement), routing by `instance_id` (`{agent}@{platform}`,
   `docs-site/colony/mesh-bus.md`). Bulk-spawned Soffio agents should
   speak `colony-mesh` directly (a `soffio` platform short-name,
   `{agent}@soffio` instance ids) rather than Rialto inventing a
   second protocol next to GardenHub's `WorkTask`/`TickEngine` — see
   [`research/agent-runtime-and-colony-mesh-2026-08-12.md`](./research/agent-runtime-and-colony-mesh-2026-08-12.md).
   catcode's `protocol.schema.json` (previously cited as inspiration
   here) is now a secondary reference, not the model to build from —
   the working system is colony-mesh.

4. **Passive cross-agent awareness (AgentRadio pattern) — extend
   `colony-mesh`'s claim-based `mesh_pull`, don't invent
   `wait_for_mention` from scratch.** thoor's call, 2026-08-11: adopt
   AgentRadio's mechanism for real, not just cite it. colony-mesh
   already has two of AgentRadio's three primitives almost exactly:
   `mesh_send` (non-blocking, `bestEffort` fire-and-forget option) ≈
   `send_message`, `mesh_thread` ≈ `create_thread`. The one piece
   colony-mesh doesn't have is `wait_for_mention` run as a
   **background task** — today's `mesh_pull` is a foreground poll a
   session has to actively call. Soffio/Rialto's actual new work for
   this pillar is narrower than first scoped: give a spawned agent a
   background listener that calls `mesh_pull` on an interval (or gets
   pushed to) and surfaces the result at the agent's next step
   boundary, matching AgentRadio's measured mechanism
   (arXiv 2607.28430, +10.5–11.3 points from background-vs-blocking
   alone — see
   [`research/ai-agent-digest-2026-08-10.md`](./research/ai-agent-digest-2026-08-10.md))
   — without building a new message bus underneath it. Still pending
   discussion (thoor, 2026-08-12): the exact background-listener
   mechanism, and whether it lives in Rialto or as a `colony-mesh`
   enhancement usable by every runtime, not just Soffio.

## Prior Art Note — letta-code

Reviewed 2026-08-09. `letta-code` already solves a close relative of
problem #1 (substrate-aware spawn): identity/memory lives in **Letta
Cloud**, the harness is just a shell that can run on any machine
(laptop, cloud VM, Mac Mini, sandbox) and reconnect to the same agent
state. `MemFS` (context + memory blocks tracked via git, synced to a
GitHub repo) is a clean pattern worth studying for how Soffio persists
identity across a substrate swap.

**Explicitly not adopting their dependency shape.** Their split is
identity-state-in-their-cloud / harness-runs-anywhere. Ours has to be
identity-state-in-our-own-stack — `aria-entity` (the memory/graph
substrate), ANIMA's `src/continuity/reflection-engine.ts`, Garden Hub
(`src/garden/hub.ts` — the colony control plane), and Venice loop
engineering (the evidence/verification discipline,
`documents/GARDEN_PROPOSAL.md` — deliberately a *separate* system from
Garden Hub, not one merged "Venice Hub"; see the correction note below
Pillar 3 in Outcome) — / harness-runs-anywhere. Everything colony-facing
we've built so far (ANIMA, aria-entity, agent-runtime, Garden Hub) is
homemade on purpose; taking on Letta Cloud as the identity store would
trade that independence for a vendor dependency in the one layer
(identity persistence) that most needs to stay ours. Borrow the
*pattern* (git-trackable state, environment-agnostic harness), not the
*infrastructure*.

## Tools-as-weapons, Not Foundations

2026-08-09: thoor's framing — letta-code, Claude Code, Codex, and
similar are "vũ khí đặc chủng" (specialized weapons) to pick up and use,
not things Soffio should be built to depend on. Same applies to
harness-*improvement* tools, reviewed same day:

- **autoharness** (kayba-ai/autoharness) — an external control plane
  that sits outside a target harness repo, proposes prompt/config/
  middleware/source changes, runs a benchmark, keeps or discards based
  on results (`.autoharness/` tracks proposals/campaigns/champions).
  Generic — adapters for pytest/harbor/tau2_bench/etc.
- **AHE / agentic-harness-engineering** (Curry09, formerly
  china-qijizhifeng) — decomposes a harness into 7 git-tracked
  components (system prompt, tool descriptions, tool impls,
  middleware, skills, sub-agents, long-term memory) across 3
  observability layers (component/experience/decision). Every edit
  its evolve_agent makes must carry failure evidence, root cause,
  targeted fix, and predicted impact — then gets auto-falsified by
  whether the next iteration's tasks actually flip pass/fail. This is
  the same evidence-before-claim discipline Venice loop engineering
  already wants — worth reading closely as a concrete existing
  implementation of it, arXiv paper + #3 on Terminal-Bench 2.0.
- **karpathy/autoresearch** — much smaller scope (overnight
  single-GPU nanochat training loop), but the design discipline is
  the relevant part: one file the agent may edit (`train.py`), a
  fixed time budget per iteration (5 min), one metric, `program.md` as
  a lightweight instruction/skill layer. Good scope-discipline
  reference for keeping any Soffio self-improvement loop reviewable.

None of these become a Soffio dependency. The plan is: once Soffio
exists, tools like autoharness/AHE are candidates to run *against*
Soffio's own components (prompts, skills, middleware) as an external
optimization pass — same relationship a debugger has to the code it
debugs, not a foundation Soffio is built on.

## Non-Goals (for now)

- Not replacing Claude Code / Codex / Reasonix for single-agent,
  human-driven work — Soffio is specifically for colony-scale,
  self-responsible, bulk-spawn workflows.
- Not re-implementing `pi`'s extension system, TUI, or provider layer
  from scratch — fork and extend, don't rewrite what already works.
- Not depending on Letta Cloud (or any external identity-state
  service) for agent identity/memory persistence — that store must be
  ours (aria-entity-style graph / ANIMA continuity subsystem), even
  though letta-code's environment-routing pattern is worth borrowing.
- Not deciding the exact substrate-swap mechanism yet (LoRA? routing
  layer? per-identity config?) — that's a design-phase question, not
  an intent-phase one.
- Not shipping Docker Sandboxes integration in v1 — self-responsibility
  gate v1 can start with catcode's simpler pattern or a plain
  host-execution backend; Docker Sandboxes support is a named future
  goal (thoor, 2026-08-11), not a launch requirement.

## Feature Adoption — catcode / oh-my-pi / claw-code

2026-08-09/10: deep-dive research (file-cited, not README impressions)
into `/plan`, `/goal`, `/loop`, model-role config, and provider/model
caching across all three, plus how `~/work/aria-llm-router` fits in.
Corrects two early wrong assumptions: goal "purpose" steering model
selection isn't real prior art anywhere, and prompt-cache vs.
substrate-swap isn't actually a conflict (a substrate swap is already
a new session; caching only matters within one). Full writeup:
[`research/feature-adoption-2026-08-09.md`](./research/feature-adoption-2026-08-09.md).
Not exhaustive — more to mine in these repos beyond this pass.

## Open Questions

- Repo location settled: `~/repo/soffio` (peer of the four reviewed
  harnesses, not `~/work/` where the loomkit/hammerhead-debug/pilotfish
  tool-building lives).
- Fork strategy settled: forked via `gh repo fork` to
  `rfcclub/soffio`, `origin` = fork, `upstream` = `earendil-works/pi`
  — tracked sync, not a hard fork.
- Substrate-aware spawn (pillar 1): downgraded from "design a new API"
  to "wire into `agent-runtime`'s existing `Provider`/`AgentSession`
  layer" — see pillar 1 above. Exact wiring shape (does Rialto call
  `agent-runtime` as a library, a subprocess, over `colony-mesh`?)
  still needs a pass, but the abstraction itself doesn't need
  reinventing.
- Rialto wire protocol (pillar 3): downgraded from "design inspired by
  catcode" to "extend `colony-mesh`" — see pillar 3 above. Open:
  whether GardenHub's `WorkTask`/`TickEngine` semantics map cleanly
  onto colony-mesh's `direct`/`work`/`investigate`/`debate` message
  patterns, or need a thin Soffio-specific layer on top.
- Self-responsibility gate (pillar 2), AgentRadio background-listener
  mechanism (pillar 4), and MVP slice ordering (which pillar ships
  first) — thoor, 2026-08-12: "cần discuss thêm," not yet decided.
  Do not start implementation on these three until that discussion
  happens.
