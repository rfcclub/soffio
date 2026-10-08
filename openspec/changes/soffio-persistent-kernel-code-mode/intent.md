# Intent: soffio-persistent-kernel-code-mode

Status: PROMOTED at Thoor's directive (Telegram, 2026-10-08 17:22–17:23 EDT). GitHub issue created 2026-10-08; GitHub issue approval pending (Thoor).

## Raw Request

Source: Aria OC dissection `~/work/soffio/research/prime-agent-tau-dissection-2026-10-08.md`, studying PrimeIntellect-ai/prime-agent (RLM pattern: one persistent Python REPL is the model's only tool; context-as-variables; `rlm.spawn(...)` for subagents; everything happens as code in a live kernel).

Thoor directive, Telegram 2026-10-08, verbatim: "Chôm ý tưởng của nó bỏ vào issue" (17:22 EDT), with scope clarification at 17:23 EDT: "Ý anh không phải kêu làm soffio nhé, mình có pi-soffio rồi, cần thì build thêm vào pi-soffio."

Implementation target: `~/work/soffio` (the pi-soffio fork). This issue is the intent of record, hosted on rfcclub/anima per colony convention. Refines the code-mode direction already present in `soffio-fabric-spike` Stage 1.

## Problem

pi-soffio's planned code mode (soffio-fabric-spike Stage 1) executes generated code, but a per-call ephemeral execution surface loses state between executions: multi-step work forces the model to re-establish intermediate state on every call, inflating tokens and latency, and context management is never a code-level operation. Prime Agent demonstrates the alternative: one long-lived REPL where context is variables, so the model manipulates its own working state programmatically instead of re-narrating it.

## Desired outcome

1. pi-soffio gains a persistent kernel (REPL-style) as its code-mode execution surface: variables and state persist across executions within a session.
2. File operations, shell commands, and existing tool calls are invocable from kernel code.
3. The kernel is inspectable by the human: current variables and state viewable at any time.
4. Kernel state is rebuildable: a kernel restart reconstructs an equivalent state from the session record.

## Non-goals

- Not a replacement for pi's agent loop — the loop stays; the kernel is an execution surface within it.
- Not a fleet or multi-identity tool (per-identity, per-session scoping only).
- No network exposure of the kernel beyond permissions pi-soffio already grants.

## Constraints

- Kernel resource limits (memory, CPU, execution time) — owner pins at implementation start.
- Kernel state is per-identity and session-scoped; no cross-identity leakage.
- A kernel crash must not corrupt or lose the session record; the session record remains the durable truth (kernel state is a cache over it).

## Success criteria

- A multi-step transformation task executes across several kernel invocations without the model re-establishing intermediate state between steps.
- Total context-token growth across the task is materially lower than the equivalent multi-tool-call chain (baseline measured at implementation start).
- Kill + restart of the kernel mid-task rebuilds an equivalent state from the session record without losing the task.

## Intent Approval

Status: colony intake at Thoor's direct directive (not colony-reviewed); GitHub issue approval pending (Thoor).
Approved by: pending (Thoor, on the GitHub issue).
Date: 2026-10-08

## Lineage

derived_from: soffio-fabric-spike (Stage 1 code-mode refinement)
sources: PrimeIntellect-ai/prime-agent (RLM pattern), dissection doc above
