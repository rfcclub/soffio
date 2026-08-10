# Soffio Evolve-Loop — Intent (Phase 2)

## Precondition — do not start before this is true

**Soffio itself must be built and running real colony work first.**
All three tools this intent draws on (autoharness, AHE, karpathy
autoresearch) evolve something against a benchmark. Before Soffio v1
exists — substrate-aware spawn, self-responsibility gate, GardenHub
wiring, see `../intent.md` — there is no real trace history to evolve
against, and evolving against a synthetic benchmark instead would
optimize for the wrong signal. This document is inert until that
precondition holds.

## Problem

Once Soffio is running the colony, its own components — system
prompts per agent role, tool descriptions, middleware, skills,
sub-agent definitions — will need to improve over time from real
failures. "Improve" without evidence is exactly what Venice loop
engineering and seal-gate exist to prevent everywhere else in this
stack; Soffio's own harness components shouldn't be the one place
that gets ad-hoc prompt tweaking instead of a real evidence-gated
process.

## Outcome

One evolve loop for Soffio's own components, combining patterns from
the three tools reviewed 2026-08-09 (full notes:
[`harness-comparison-2026-08-09.md`](./harness-comparison-2026-08-09.md)
covers the harness-choice research; the tools-as-weapons framing is in
`../intent.md`):

1. **From AHE — component decomposition + evidence-per-edit.**
   Borrow the 7-component split (system prompt, tool descriptions,
   tool implementations, middleware, skills, sub-agents, long-term
   memory), git-tracked, applied to Soffio's own harness. Every
   proposed change carries four fields — failure evidence, root
   cause, targeted fix, predicted impact — and gets falsified or
   confirmed by whether the next iteration's real tasks actually flip
   pass/fail. This is the concrete mechanism for applying Soffio's
   self-responsibility gate *recursively to Soffio itself*, not just
   to the agents it spawns.

2. **From autoharness — campaign/workspace/track/champion bookkeeping.**
   Proposals, iterations, and champion state need a persisted shape
   (their `.autoharness/` is the reference) so an evolve run is
   resumable and auditable, not a one-shot script. Adopt the
   bookkeeping shape, not the tool itself.

3. **From karpathy/autoresearch — scope discipline.** One bounded
   component edited per iteration, a fixed evaluation budget, a
   lightweight instruction file (their `program.md`) as the
   equivalent of Soffio's evolve-agent brief. Diffs stay reviewable
   because scope stays small — this is the guardrail against the
   "god-file" failure mode all four harnesses in the comparison
   research showed to varying degrees.

## The eval signal is ours, not theirs

This is the one place Soffio's evolve loop cannot just copy the
source tools: AHE evaluates against Terminal-Bench/SWE-bench,
autoresearch against val_bpb on a fixed dataset — external,
synthetic benchmarks. Soffio has no equivalent synthetic benchmark to
borrow, and shouldn't invent one. The eval signal has to be the
colony's own real operational history: GardenHub task completion
rate, Venice loop engineering's streak/holdout results, seal-gate
PASS/REVISE/BLOCK rates over time. An edit to Soffio's own prompts or
skills is judged by whether it measurably improves *real* colony
throughput, not a proxy benchmark.

## Non-Goals

- Not adopting AHE's Agent Debugger infrastructure directly — their
  README states it's only partially open-sourced. If a
  trace-digestion tool is needed later, build a small one against our
  own trace format rather than depending on theirs.
- Not standing up E2B/harbor-style sandbox infrastructure — Soffio's
  own self-responsibility-gate sandbox (catcode-inspired, per
  `../intent.md`) is the execution environment; a separate benchmark
  sandbox would be a second thing to maintain for no real gain.
- Not starting any implementation of this loop before Soffio v1 ships
  and has accumulated real colony task history — see Precondition.

## Open Questions

- How much real operational history is "enough" before the evolve
  loop can run safely, without overfitting to a small early trace
  set?
- Autonomous (autoharness-style, runs overnight unattended) or
  human-gated per Venice loop engineering's existing hill-climbing
  discipline (`../../../documents/GARDEN_PROPOSAL.md`)? Leaning
  human-gated at first, autonomous only once the gate itself has a
  track record — but not decided.
