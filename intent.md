# Soffio — Intent

## Name

**Soffio** (Italian, "breath" — as in *soffio di vita*, "breath of life").
`pi` (earendil-works) is the body being forked; the substrate (whichever
model backs an agent — Claude, Kimi, DeepSeek, Gemini, ...) is the soul.
Soffio is what connects the two and keeps it alive across substrate
changes.

**Rialto** — the orchestrator/hub component inside Soffio (named after
Venice's original center, "Rivoalto" — the hub the city grew from).
Ties into ANIMA's existing Venice loop engineering naming
(`documents/GARDEN_PROPOSAL.md`) and `src/garden/hub.ts` (GardenHub).

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

A fork of `pi` — Soffio — that adds three things `pi` doesn't have out
of the box:

1. **Substrate-aware spawn.** Spawn N agents (up to all 16 in
   `data/colony.json`) where each agent is an *identity*, not a
   *provider call*. The identity can run on Claude, Kimi, DeepSeek,
   Gemini, etc., and switching the underlying model must not break
   continuity — same pattern already proven for Aria individually
   (`~/alaya/`, "substrate rented, identity owned"), generalized to a
   whole colony.

2. **Self-responsibility gate.** `pi` ships with no built-in
   permission/sandbox system by design (their choice, documented in
   `docs/containerization.md`) — fine for a single trusted developer,
   not fine for 16 autonomously-spawned agents. Soffio's gate is
   evidence-based, not a confirm-dialog: no "done" claim survives
   without the kind of proof Venice loop engineering already defines
   (CLI memory recall, mechanical verification, streak loops, blank-arm
   holdout tests, human-gated hill-climbing).

3. **GardenHub wiring.** Bulk-spawned agents need to speak to
   `src/garden/hub.ts` (`GardenHub`, `WorkTask`) and
   `src/garden/tick-engine.ts` natively, not bolt on as an external CLI
   that happens to run alongside ANIMA. Rialto (the orchestrator inside
   Soffio) is the transport that speaks GardenHub's protocol —
   inspired by catcode's one-protocol-many-runtimes design
   (`protocol/protocol.schema.json`), which was the cleanest boundary
   pattern of the four reviewed.

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
substrate), ANIMA's `src/continuity/reflection-engine.ts`, and Venice
Hub — / harness-runs-anywhere. Everything colony-facing we've built
so far (ANIMA, aria-entity, agent-runtime, Venice Hub) is homemade on
purpose; taking on Letta Cloud as the identity store would trade that
independence for a vendor dependency in the one layer (identity
persistence) that most needs to stay ours. Borrow the *pattern*
(git-trackable state, environment-agnostic harness), not the
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

## Open Questions

- Repo location settled: `~/repo/soffio` (peer of the four reviewed
  harnesses, not `~/work/` where the loomkit/hammerhead-debug/pilotfish
  tool-building lives).
- Fork strategy: hard fork with our own remote, or fork + tracked
  upstream sync (like oh-my-pi does from pi-mono, which this session's
  review flagged as a real ongoing maintenance tax)?
- Exact shape of the "substrate-aware spawn" API — needs a design pass
  against `pi`'s existing provider abstraction
  (`packages/ai`) before committing to an approach.
