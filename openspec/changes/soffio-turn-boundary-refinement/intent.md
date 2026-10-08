# Intent: soffio-turn-boundary-refinement

Status: PROMOTED at Thoor's directive (Telegram, 2026-10-08 17:22–17:23 EDT). GitHub issue created 2026-10-08; GitHub issue approval pending (Thoor).

## Raw Request

Source: Aria OC dissection `~/work/soffio/research/prime-agent-tau-dissection-2026-10-08.md`, studying PrimeIntellect-ai/prime-agent (Rust; RLM + Continual Harness, crates/pa-daemon + skills/refine) and huggingface/tau (Python; session-as-tree).

Thoor directive, Telegram 2026-10-08, verbatim: "Chôm ý tưởng của nó bỏ vào issue" (17:22 EDT), with scope clarification at 17:23 EDT: "Ý anh không phải kêu làm soffio nhé, mình có pi-soffio rồi, cần thì build thêm vào pi-soffio" — the idea targets additions to the existing pi-soffio fork, not a new soffio buildout.

Implementation target: `~/work/soffio` (the pi-soffio fork). This issue is the intent of record, hosted on rfcclub/anima per colony convention.

## Problem

pi-soffio's core promise is durable residency: an identity keeps working context alive across substrate swaps. But today, operational learnings an identity develops while working — recurring failures, reusable tactics, delegation roles, behavior policies — evaporate at session end. The substrate swap preserves the model layer, not the accumulated operational state: every new session starts blank, and every substrate swap resets working context. An identity that worked with pi for weeks carries nothing forward except what a human manually copied into its instructions.

## Desired outcome

1. A durable supplemental-state store per identity in pi-soffio (the Rialto layer): supplemental prompts, memories, skill descriptions, and reusable task/subagent specifications — kept strictly separate from the immutable base instructions.
2. Updates are small and evidence-backed, applied only at turn boundaries (never mid-turn); applying an update rebuilds the effective instruction set and the session resumes automatically.
3. Two stores: session-scoped (default) and global (cross-session). Promoting a local entry to global is an explicit, logged action.
4. Every store change is snapshotted; rollback to any prior snapshot is supported.
5. Substrate swaps preserve the store: the supplemental state belongs to the identity, not to the model backing it.

## Non-goals

- No behavior-policy automation and no affect/mood state — operational learnings only (the colony's separate emotion work is out of scope here).
- No colony-mesh or Garden Hub integration changes.
- No new verification gate: proposed updates reuse pi-soffio's existing review/verify patterns.
- No work outside pi-soffio.

## Constraints

- Single-writer discipline per identity store; every change carries provenance (source turn, evidence reference).
- Store format versioned from day one.
- Implementation stays inside pi-soffio's existing architecture (Rialto + pi's own agent loop and provider layer); no new runtime.
- The immutable base instruction set must remain byte-identical through any refinement (verifiable by hash).

## Success criteria

- A recurring failure observed in one turn can be refined into the store at the next turn boundary, survives a substrate swap, is present in a later session, and can be rolled back via snapshot.
- The base instruction file is byte-identical after any refinement.
- Global promotion is logged and reversible.
- Owner pins at implementation start: the exact store format and the retention policy.

## Intent Approval

Status: colony intake at Thoor's direct directive (not colony-reviewed); GitHub issue approval pending (Thoor).
Approved by: pending (Thoor, on the GitHub issue).
Date: 2026-10-08

## Lineage

derived_from: soffio-fabric-spike (Stage 1 durable-residency refinement)
sources: PrimeIntellect-ai/prime-agent (crates/pa-daemon, skills/refine), dissection doc above
