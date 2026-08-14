# Soffio — Intent

## What Soffio Is (read this first)

**Soffio is an agent harness, like the Claude Code CLI or the Claude
Agent SDK — reusable, not tied to any one identity.** thoor's
analogy, 2026-08-13: Claude Code is a CLI/harness; the Claude Agent
SDK lets you spin up many agents each carrying Claude Code's power.
Soffio is the same shape: **one Soffio process + one identity = one
agent.** Running Coda through Soffio and running Rina through Soffio
are two separate Soffio processes, each paired with its own identity
— not one Soffio managing sixteen.

**Soffio is not a fleet manager.** It doesn't spawn a roster of
identities, doesn't track who's doing what across the colony, doesn't
decide who works on which task. That's **Garden Hub**'s job
(`src/garden/hub.ts`, ANIMA's real colony control plane), not
Soffio's — Soffio doesn't compete with it or duplicate it.

**Multiple Soffio-identity agents work together as peers**, not
through a Soffio-side orchestrator — thoor's reference point is how
Craft Build coordinates multiple independent workers, not a
master/worker hierarchy. Peer coordination happens over `colony-mesh`
(ANIMA's existing cross-runtime message bus), the same bus Garden
Hub's own agents already use.

**Soffio** the name: Italian for "breath" (*soffio di vita*, "breath
of life"). `pi` (earendil-works) is the body being forked; the
substrate (whichever model backs the identity — Claude, Kimi,
DeepSeek, Gemini, ...) is the soul. Soffio connects the two and keeps
the connection alive across a substrate change.

**Rialto**, inside each Soffio process: the small join-client +
local bookkeeping layer for that one agent (see Outcome §1). Named
after Venice's "Rivoalto," the hub the city grew from — but Rialto
itself is not a hub in the fleet sense; it's scoped to the one agent
its Soffio process is running.

## The four things Soffio actually does, per agent

1. **Join, don't spawn a runtime.** Each identity already owns its own
   named runtime (`coda-runtime` today; `aria-runtime`, `rina-runtime`,
   etc. as they get built — one runtime per identity, a deliberate
   convention, not something Soffio adds a second layer over). Soffio
   doesn't run an AI loop itself — it joins the identity's own runtime
   (`POST /v1/join`, the same protocol `coda-runtime` already
   implements) to obtain that identity's substrate, then runs the
   actual chat/tool-execution turn using **`pi`'s own agent loop**
   (`packages/agent/src/agent.ts` + `agent-loop.ts`), inherited whole
   from the fork — not ANIMA's `AgentSession` (that's ANIMA's own
   loop, for ANIMA's own agents, unrelated to Soffio). Model/substrate
   swaps happen entirely inside `pi`'s own provider layer — no new
   API needed there either.

2. **Prove it, don't just say it — by wiring in `seal-gate`, not
   building a new gate.** thoor, 2026-08-13: we already have
   `loomkit`/`seal-gate`/`hammerhead-debug`/`pilotfish` as a real,
   running foundation — pillar 2 shouldn't reinvent what they already
   do. Mapped onto Martin Fowler's harness-engineering vocabulary
   (`research/ai-agent-digest-2026-08-10.md`'s companion read,
   2026-08-13): `seal-gate` already IS the sensor layer — computational
   (deterministic PASS/REVISE/BLOCK) with an optional inferential mode
   (`--llm`), exactly the "cheap/fast by default, expensive/judgment
   layer optional" split the article argues for. Soffio's identity, on
   a "done" claim, calls `seal-gate` the same way this very session's
   Claude Code hooks already do — not a new gate, a wiring task.
   `loomkit` is the guides layer (spec/plan as checked-in artifacts,
   matching the OpenAI Codex team's "AGENTS.md as table of contents,
   not encyclopedia" lesson from the same read) — an identity doing
   real work under Soffio should follow `loomkit`'s workflow, not
   invent its own task tracking. `hammerhead-debug` is what the
   identity reaches for when a test won't go green (hypothesis-gated,
   no fix without a confirmed observation). `pilotfish` is available
   if driving the identity through `plan.json` tasks with retry is
   useful for a given workflow. All four stay "tools-as-weapons" per
   the section below — called into, not forked into Soffio's own code.
   The execution sandbox itself can still lean on an existing pattern
   (catcode's microVM backend, or Docker Sandboxes as a named future
   option) rather than being hand-rolled.

3. **Speak `colony-mesh` — mostly already done, not a Soffio build.**
   Corrected 2026-08-14: `coda-runtime`'s `join()` already registers
   the joined instance on `colony-mesh` **automatically, on every
   join, unconditionally** — live spec:
   `openspec/specs/coda-runtime-mesh-registration/spec.md` (in the
   `anima` repo). `instance_id` derives deterministically from
   `(agentId, surfaceId)` — a distinct `surfaceId` per Rialto join
   yields a distinct, independently-addressable mesh recipient with no
   extra registration step. **Soffio's own work for this pillar is
   verification, not implementation**: confirm a Rialto-initiated join
   (via Track A) gets a working mesh address for free, using the
   existing `mesh_send`/`mesh_pull`/`mesh_ack`/`mesh_thread` tools —
   the same bus every other runtime in the colony already uses. No
   second message bus, and — corrected from the earlier draft — no
   new registration code either.

4. **Listen while working — already built, use it.** Corrected
   2026-08-14: `colony-mesh` already has a real push/wakeup layer, not
   just poll — archived spec:
   `openspec/archive/colony-mesh-realtime-pubsub-*/specs/mesh-realtime-pubsub/spec.md`
   (in the `anima` repo). Publish persists to SQLite first, *then*
   emits a wakeup to registered listeners (multiple simultaneous
   listeners supported, direct/broadcast/topic routing, listener
   expiry, wakeup failure never fails publish). SQLite stays the
   source of truth — a wakeup is a hint to go read, not the delivery
   itself. This already covers AgentRadio's `wait_for_mention`
   primitive (plus the two — `mesh_send` ≈ `send_message`, `mesh_thread`
   ≈ `create_thread` — already noted). **Soffio's work here shrinks to:
   register a listener for the joined identity and surface its wakeups
   at the agent's next step boundary** — no poll loop to build, the
   push mechanism already exists.

## Problem

We already have general-purpose harnesses (Claude Code, Codex,
Reasonix) and reviewed four "coding agent CLI" forks/relatives
(claw-code, oh-my-pi, pi, catcode). None of them fit what one colony
identity needs when run headlessly: they assume a human at the
keyboard, one fixed provider, and (at most) a generic
permission/sandbox model built for a single trusted developer — not
an identity that must survive a substrate swap, prove its own work,
and coordinate with siblings over a shared bus.

We picked `pi` (earendil-works) as the fork base after a 4-way review:
strongest extension API, and — uniquely among the four — it reads
Claude Code's and Codex's skill directories directly, so an identity
running under Soffio isn't locked into one vendor's skill format. Full
review notes: [`research/harness-comparison-2026-08-09.md`](./research/harness-comparison-2026-08-09.md).

## Non-Goals

- **Not a fleet manager.** Spawning/tracking/routing across many
  identities is Garden Hub's job. Soffio runs one agent per process.
- Not replacing Claude Code / Codex / Reasonix for human-driven work —
  Soffio is specifically for a colony identity run headlessly,
  self-responsibly.
- Not re-implementing `pi`'s extension system, TUI, provider layer, or
  agent loop from scratch — fork and extend, don't rewrite what
  already works.
- Not depending on Letta Cloud or any external identity-state service
  — identity/memory persistence stays in our own stack (`aria-entity`,
  ANIMA's continuity subsystem). See
  [`research/agent-runtime-and-colony-mesh-2026-08-12.md`](./research/agent-runtime-and-colony-mesh-2026-08-12.md)
  for why (letta-code's environment-routing *pattern* is worth
  studying; their cloud dependency is not).
- Not deciding the exact substrate-swap mechanism yet — that already
  lives inside `pi`'s own provider layer; nothing new to design.
- Not shipping Docker Sandboxes integration in v1 — named future goal,
  not a launch requirement.
- Harness-improvement tools (autoharness, AHE, karpathy/autoresearch —
  see [`research/ai-agent-digest-2026-08-10.md`](./research/ai-agent-digest-2026-08-10.md))
  are candidates to run *against* Soffio later, never a dependency
  Soffio is built on.
- Not reinventing `loomkit`/`seal-gate`/`hammerhead-debug`/`pilotfish`
  — see pillar 2 above. These are called into as running tools (same
  "tools-as-weapons" relationship as Claude Code/Codex/Reasonix), not
  forked into or duplicated by Soffio's own code.

## Current Status (2026-08-14)

No code written yet. LoomKit spec+design exist for two slices —
`openspec/changes/track-a-rialto-spawn/` (join a runtime, run `pi`'s
loop) and `openspec/changes/track-b-colony-mesh-participation/`
(verify mesh registration + wakeup already work for a Rialto join) —
see [`plan-mvp-slice.md`](./plan-mvp-slice.md) for the dependency
graph. thoor's steer: go slowly, one small vertical slice (Coda + one
task) before touching anything else.

**Prerequisite outside Soffio's own scope, thoor 2026-08-14**:
identity runtimes need to be more complete before Soffio can go beyond
Coda. Confirmed by reading current code/specs, not assumed:
`coda-runtime` is the only identity runtime that's a real running
process today; its `recall` still returns `409 unimplemented` (a
deliberate first-phase limitation per
`openspec/specs/identity-runtime-web-service/spec.md`, not an
oversight); a generic per-agent path exists
(`~/.identityRuntime/{agentId}/CORE_IDENTITY`, symlinked to a
compiled `CORE_IDENTITY.md`) but is only exercised for Coda so far.
thoor's bar: at least 3 identity runtimes complete with working
`remember` *and* `recall`, all 16 eventually resolving
`CORE_IDENTITY`. This is ANIMA-repo work, upstream of and outside
Soffio's own scope — tracked here as a dependency, not something
Soffio's own tracks build.

**Not dropped, just not yet tracked**: the concrete features mined
from oh-my-pi/catcode/claw-code
([`research/feature-adoption-2026-08-09.md`](./research/feature-adoption-2026-08-09.md))
are real and still the plan — they just don't have a LoomKit track
yet because they land on pillar 2 (self-responsibility gate) and
parts of pillar 1 (provider config) that are still "cần discuss
thêm," not on Track A/B's narrow MVP scope. Concretely, still queued:
- oh-my-pi's `/plan` mode (sandbox-write-guard), `/loop` mode (maps to
  Venice loop streak cycles), named-role model table, model-cache
  fingerprint-check pattern.
- catcode's `/goal` phase state machine + step DAG (closest existing
  shape to GardenHub `WorkTask` coordination — relevant once Track B
  needs more than a chatter-only thread), `RoleModels` explicit
  override, `PROVIDER_PRESETS` pattern.
- claw-code's `PermissionMode` ordinal-gating primitive (pillar 2), and
  its disk-flag-only plan-mode bug as a negative lesson (mode toggles
  must flip in-memory state same-turn, not on next restart).
- `aria-llm-router` as the provider registry/router Soffio should
  reuse rather than rebuild.

**Process fix, thoor 2026-08-14**: today's Track B rework
(re-discovering colony-mesh's wakeup layer and `coda-runtime`'s
auto-registration from scratch, by grepping `openspec/`) is exactly
the kind of durable finding that `~/agora/hive` — a real, working,
nearly-unused shared lexicon CLI (`hive publish`/`search`/`list`) —
exists to prevent from being rediscovered next time. Published the
three real findings from this session
(`colony-mesh-has-realtime-wakeup-not-poll-only`,
`coda-runtime-join-auto-registers-colony-mesh`,
`identity-runtime-recall-unimplemented-by-design`) to `hive` so a
future session — Soffio-related or not — can `hive search
colony-mesh` before re-deriving this from source. Going forward:
check `hive search` before a fresh research pass on ANIMA-side
internals, not after.

## Open Questions

- Track A: confirm the MVP slice targets Coda specifically (only
  `coda-runtime` exists today); how `CodaJoinResponse`'s substrate
  payload feeds into `pi`'s agent loop's system/context input.
- Track B: where `platform: "soffio"` registration actually happens
  (likely `coda-runtime`'s own launch config, an ANIMA-repo change,
  not a Soffio-side one); whether
  `openspec/specs/colony-mesh-mcp-runtime-usage/spec.md` needs a
  `soffio` section.
- Self-responsibility gate specifics (§2) and the background-listener
  mechanism (§4) — thoor: "cần discuss thêm," not yet decided. Do not
  start implementation on these until that discussion happens.

## Revision History

Kept short on purpose — full reasoning for each correction lives in
git history and the `research/` files linked above, not repeated here.

- **2026-08-13**: Reframed from "Soffio bulk-spawns 16 agents" to
  "one Soffio process per identity, peers coordinate via colony-mesh,
  Garden Hub owns fleet-level roster/routing." Corrected Track A's
  design: Rialto uses `pi`'s own agent loop, not ANIMA's
  `AgentSession` (that was a real error, not a naming nit). Dropped
  the invented name "Venice Hub" — Garden Hub (control plane) and
  Venice loop engineering (evidence discipline) are deliberately
  separate systems.
- **2026-08-12**: Found ANIMA already has a per-identity "identity
  runtime" pattern (`coda-runtime`) and a working cross-runtime
  message bus (`colony-mesh`) — narrowed Track A and Track B from
  "design something new" to "integrate with what exists."
- **2026-08-11**: Adopted the AgentRadio background-listening pattern
  and named Docker Sandboxes as a future gate backend.
- **2026-08-09/10**: Picked `pi` as the fork base after a 4-way
  harness review; mined catcode/oh-my-pi/claw-code for adoptable
  features; corrected two early wrong assumptions (goal "purpose"
  steering model selection; prompt-cache vs. substrate-swap "conflict"
  — neither was real).
