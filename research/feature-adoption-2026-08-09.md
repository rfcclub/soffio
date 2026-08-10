# Feature Adoption — catcode, oh-my-pi, claw-code

Deep-dive research 2026-08-09/10, cited to real files, not README
impressions. Goal: figure out which concrete mechanisms from these
three repos are worth adopting into Soffio (fork of `pi`), and correct
two assumptions that turned out wrong before they got baked into a
design.

Not exhaustive — thoor's note stands: there's more in these repos than
what's covered here, this is what got dug into this pass.

## Correction: "goal has a purpose that steers model/role" — not true in either repo

Neither `catcode` nor `oh-my-pi` does this. Recording precisely so
Soffio doesn't inherit a wrong assumption:

- **oh-my-pi** `Goal` (`goals/state.ts:6-14`) has exactly:
  `id, objective, status, tokenBudget, tokensUsed, timeUsedSeconds,
  createdAt, updatedAt`. No `purpose`, no `role`, no `model` field.
  The only role/model coupling anywhere in `goals/` is
  `guided-setup.ts:69` calling `resolveRoleModelWithThinking("slow")`
  — a fixed role for running the *interview UI* that helps a user
  draft an objective, unrelated to the goal's content.
- **catcode** `StartGoalArgs` (`core/src/goal.rs:554-571`) has
  `goal: String` (free text) + `role_models: RoleModels` +
  `allowed_models`/`allowed_providers`. `RoleModels`
  (`goal.rs:245`) is exactly 3 fields: `planner`, `worker`,
  `reviewer` — **explicit**, set by the caller, not inferred from
  parsing the goal text. There is no "purpose" field either.

**What this means for Soffio**: "attach a purpose string and have the
system infer the right model/role" isn't proven prior art anywhere —
it'd be new ground, not adoption. The proven pattern is catcode's:
an explicit `role_models` override map passed at goal-start time.
Cheaper, testable, no inference step to get wrong. Recommend Soffio
copy the *explicit override* shape, not invent purpose-inference.

## /goal mode — the real state machine to borrow: catcode's

`catcode`'s `GoalPhase` enum (`core/src/goal.rs:31`):
`Idle → Planning → [Reviewing] → PlanReady → Deploying → Running →
Synthesizing/Verifying → Done/Failed`. `GoalPlan`/`GoalStep`
(`goal.rs:203-224`): each step carries `agent`, `task`, optional
`model` override, `depends_on` (a real DAG, not a flat task list).
Role-model resolution order per step: step's own `model` → allowlist
→ parent default (`goal.rs:242-244`).

This is a more complete goal lifecycle than oh-my-pi's (which is just
`objective` + `token_budget` + status, no phases, no step DAG). If
Soffio needs a `/goal` equivalent for coordinating GardenHub
`WorkTask`s, catcode's phase machine + step DAG is the one to study —
it already maps closely onto what `WorkTask` dependency tracking would
need.

## /plan mode — oh-my-pi's is the most mature, claw-code is a cautionary tale

**oh-my-pi** (`tools/plan-mode-guard.ts`): not a restricted toolset —
same tools stay available, but every write/edit/rename/delete against
the real working tree gets intercepted (`enforcePlanModeWrite()`,
L134-155) and thrown unless the target resolves inside a `local://`
artifact sandbox. Plan gets written to `local://<slug>-plan.md`,
protected from context-compaction pruning
(`plan-mode/plan-protection.ts`), and only handed off to subsequent
turns/subagents *after* explicit user approval
(`plan-mode/approved-plan.ts`, `plan-handoff.ts` —
`loadOverallPlanReference()` is gated off during plan mode itself, so
an unapproved draft can never leak as "the plan"). This is the
implementation to borrow — it's already inside the codebase Soffio
forked from.

**claw-code** (`runtime/src/permissions.rs`,
`rust/crates/tools/src/lib.rs:769-787`): plan mode is a *persistent
settings.json flag*, not live session state. `EnterPlanMode`/
`ExitPlanMode` write `settings.json` + a `PlanModeState` sidecar file,
but that write never propagates into the already-constructed
in-memory `PermissionPolicy` — the mode change only takes effect on
the *next process start*. Confirmed by grep: neither tool is wired
back into the live policy anywhere in `main.rs`. Worse, since both
tools require `WorkspaceWrite` but plan mode's active mode is
`ReadOnly`, calling `ExitPlanMode` *while in plan mode* falls into the
Deny branch unless an explicit allow-rule exists (none found) — a real
UX gap vs. real Claude Code, where exiting plan mode is a prompted,
in-turn transition, not a restart.

**Lesson for Soffio**: whatever plan-mode gate gets built, the
exit transition must flip the *in-memory* policy for the rest of the
current turn loop. Do not repeat claw-code's disk-flag-only mistake.
Worth borrowing regardless: claw-code's `PermissionMode` enum with
`Ord`-based comparison (`permissions.rs:8-15`) — a per-tool
`required_mode`, active session mode compared by ordinal — is a clean,
reusable gating primitive, independent of the broken wiring around it.

## /loop mode — oh-my-pi, simple and worth adopting cheaply

`builtin-registry.ts:288-306` → `interactive-mode.ts:1277`. `/loop
[count|duration] [prompt]` captures the next submitted prompt (or an
inline one) as `loopPrompt`, sets `loopModeEnabled=true`. After each
turn, an 800ms deferred timer (`#deferLoopAutoSubmit`, L1149 — "brief
delay so the user has a chance to press Esc between iterations")
re-submits the same prompt text. `loop.mode` setting picks the
per-iteration action: `"prompt"` (default), `"compact"` (compacts
first), or `"reset"` (clears first). Exit: iteration/duration limit
exhaustion, or Esc (`pauseLoop()`, drops the prompt but keeps the mode
armed for the next manual submit). No cross-iteration checkpointing
beyond conversation history — it's literally "resend this string N
times / for T duration."

**Relevance to Soffio**: this is close to the mechanical primitive
Venice loop engineering's streak loops need — a bounded, resumable,
same-prompt repetition with an explicit stop condition. Cheap to
adopt as-is; the `compact`/`reset` per-iteration variants map
naturally onto "verify again fresh" vs. "verify again with full prior
context" streak styles.

## Model roles — different shapes in each repo, oh-my-pi's names are the readable default

- **oh-my-pi**: named tiers (`default`, `smol`, `slow`, `task`,
  `advisor`, `tiny`), config in `~/.omp/agent/settings.yml`
  `modelRoles`. See `OMP_HUONGDANSUDUNG.md` for the full breakdown —
  already documented, no need to re-derive.
- **catcode**: no unified named-role enum. Binary `fast_roles`/
  `strong_roles` matched by agent name (`RoutingConfig`,
  `config.rs:399-412`) plus `fast_markers`/`strong_markers` substring
  heuristics on model id (`score_model()`, `config.rs:466`), separate
  `AdvisorConfig` (`config.rs:311-321`: `model` + `subagent_model`)
  and `VisionConfig` (`vision.rs:18-21`: `vision_models` +
  `vision_model`) as their own standalone configs, plus goal-scoped
  `RoleModels` (planner/worker/reviewer) on top of all that.

**For Soffio**: oh-my-pi's flat named-role alias table is the better
starting shape for the "multiple models assigned to the same role,
spawn N agents with different models under concurrency/strategy"
idea thoor described — a role already resolves to *one* model id
today in both repos; extending it to resolve to a *list* with a
selection strategy (round-robin, all-N-concurrently,
cheapest-first-with-fallback) is a small, well-scoped change to an
existing table rather than a new subsystem.

## Model-cache / model-resolve — two different caches, don't conflate them

Both repos cache **model catalog metadata** (context window, pricing,
capability flags) — neither of these is LLM **prompt-content**
caching (Anthropic `cache_control` breakpoints, etc.). Keep these
concerns separate in Soffio's design; see the prompt-cache section
below for the actually-separate problem.

- **oh-my-pi**: SQLite (`bun:sqlite`, `packages/catalog/src/
  model-cache.ts`, table `model_cache` keyed by `provider_id`).
  `CACHE_SCHEMA_VERSION = 8` — every bump wipes stale rows via a
  version-mismatch `DELETE`; 8 historical invalidation reasons are
  documented in the comment trail (thinking-mode shape changes,
  effort-tier collapsing, etc.). Resolution
  (`model-manager.ts resolveProviderModels()`, L110-208): cache read →
  `staticFingerprint` hash check against the bundled static catalog →
  fresh+fingerprint-match+authoritative → return cached merge directly
  (~800ms cold-start win, documented). Otherwise fetches models.dev +
  provider-specific dynamic endpoints in parallel, merges
  static+modelsdev+cache+dynamic, dedupes effort-tier variants
  (`collapseBuiltModelVariants`), writes back. `ModelRefreshStrategy`
  = `"online" | "offline" | "online-if-uncached"`.
- **catcode**: flat-file, `core/src/models_dev.rs`. 24h disk TTL
  (`MODELS_DEV_CACHE_TTL = 86400`). `fetch_models_dev()`: disk cache →
  live fetch (10s timeout) → write disk cache; non-fatal on failure,
  falls back to stale cache then curated defaults. `lookup_entry()`
  does a 3-tier match: direct full id → provider-slug-prefixed (host→
  slug map, e.g. `api.deepseek.com`→`"deepseek"`) → suffix match.
  Explicitly an **overlay**: only fills fields the curated table and
  live provider API left generic — curated + live always win over
  models.dev.
- **Provider-add**: catcode ships `PROVIDER_PRESETS`
  (`config.rs:672-690`) — built-in vendor templates (base_url +
  api_key_env known ahead of time) so adding a provider is "pick a
  preset, paste a key" via `/login`/`/provider`, not hand-editing
  JSON. oh-my-pi's equivalent is the `models.yml` config path plus the
  3-file built-in-provider path documented in
  `OMP_HUONGDANSUDUNG.md` §8.

## aria-llm-router — reuse instead of rebuild

`~/work/aria-llm-router` already exists and is homemade (fits the
"hàng nhà mình" rule from `../intent.md`'s letta-code note): a
transparent Anthropic API proxy with local/Haiku/Sonnet/DeepSeek
routing tiers, `src/providers/` (`provider-registry.ts`,
`provider-router.ts`, `provider-loader.ts`, OpenAI↔Anthropic
translation layers), OAuth flow, context compaction, and a cost
dashboard. This is a real candidate to *be* (or back) Soffio's
provider-add/model-resolve layer instead of reimplementing catcode's
`PROVIDER_PRESETS` or oh-my-pi's `model-cache.ts` from scratch —
worth a design pass on whether Soffio's model-role table calls into
aria-llm-router as its resolution backend rather than duplicating
provider config plumbing a third time.

## Design note — prompt-cache vs. substrate swap: not actually a conflict

Originally flagged this as an open problem (substrate swap invalidates
provider-scoped prompt cache, so substrate-aware spawn and prompt
caching seemed to need reconciling). thoor's correction, 2026-08-10:
**a substrate swap already IS a new session** — every run is its own
session regardless. Prompt caching (Anthropic `cache_control`
breakpoints, OpenAI automatic prefix caching, etc.) only ever matters
*within* a session, turn-to-turn, and a session is already pinned to
one substrate for its whole lifetime. There's no case where a live
session's cache needs to survive a substrate change, because a
substrate change doesn't happen mid-session — it happens as a new
session starts. Nothing to reconcile.

What's still worth keeping simple (not a hard requirement, just good
practice): a stable prompt-layering order within a session — identity
layer, then role/skill layer, then task/context layer, most stable
first — since that's ordinary cache-hit hygiene within any single
substrate, same as it would be for a single-substrate harness. No
per-provider research pass needed before this; it falls out of normal
prompt assembly, not a special substrate-aware mechanism.

## Summary — what to actually adopt

| From | What | Why |
|---|---|---|
| oh-my-pi | `/plan` mode implementation (sandbox-write-guard + approval + compaction-protected plan file) | Most mature of the three, already in the forked codebase |
| oh-my-pi | `/loop` mode implementation | Cheap, maps onto Venice loop streak cycles |
| oh-my-pi | Named-role alias table shape (`modelRoles`) | Best starting shape for multi-model-per-role spawn |
| oh-my-pi | Model-cache fingerprint-check pattern (skip refetch when nothing changed) | Real ~800ms win, cheap to replicate |
| catcode | `/goal` phase state machine + step DAG (`GoalPhase`, `GoalStep.depends_on`) | Closest existing thing to what GardenHub `WorkTask` coordination needs |
| catcode | Explicit `RoleModels` override shape (planner/worker/reviewer) | Proven alternative to purpose-inference, which isn't proven prior art anywhere |
| catcode | `PROVIDER_PRESETS` pattern | Lowers friction for adding providers without hand-edited JSON |
| claw-code | `PermissionMode` ordinal-gating primitive | Clean, independent of the broken wiring around it in that repo |
| claw-code | The disk-flag-only plan-mode bug | Negative lesson: mode toggles must flip in-memory policy same-turn |
| aria-llm-router | Provider registry/router as a whole | Don't rebuild what's already homemade and working |
