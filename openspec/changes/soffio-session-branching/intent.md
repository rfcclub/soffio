# Intent: soffio-session-branching

Status: PROMOTED at Thoor's directive (Telegram, 2026-10-08 17:22–17:23 EDT). GitHub issue created 2026-10-08; GitHub issue approval pending (Thoor).

## Raw Request

Source: Aria OC dissection `~/work/soffio/research/prime-agent-tau-dissection-2026-10-08.md`, studying huggingface/tau (Python; session-as-tree with branching: durable JSONL sessions where any prior point can branch and proceed independently; `session/tree.py`, `session/jsonl.py`).

Thoor directive, Telegram 2026-10-08, verbatim: "Chôm ý tưởng của nó bỏ vào issue" (17:22 EDT), with scope clarification at 17:23 EDT: "Ý anh không phải kêu làm soffio nhé, mình có pi-soffio rồi, cần thì build thêm vào pi-soffio."

Implementation target: `~/work/soffio` (the pi-soffio fork). This issue is the intent of record, hosted on rfcclub/anima per colony convention.

## Problem

pi-soffio sessions are linear. When an experiment path fails mid-session, the working prefix is lost or polluted: the identity either continues from a bad state or restarts the session and loses everything before the failure. Explore-then-commit workflows — trying one implementation path, keeping another — have no first-class support. Tau demonstrates session-as-tree: a durable session where any prior point can branch and proceed independently, with the main line preserved.

## Desired outcome

1. pi-soffio sessions support branching at any turn: a branch forks the conversation state at that point and proceeds independently.
2. Branches are listable and switchable; switching restores exactly the state at the branch point.
3. Learnings can be carried between branches explicitly (the identity can read another branch's outcomes); transcript merge is not automatic.
4. Session storage uses structural sharing so a branch does not duplicate the full transcript on disk.

## Non-goals

- No automatic merging of divergent file changes (branching is conversation-state, not git).
- No colony-mesh or Garden Hub changes.
- No GUI beyond minimal CLI affordances (list, switch, branch).

## Constraints

- Branch identity is stable across substrate swaps: a branch survives an engine change (soffio's core promise applies to branches too).
- Retention and pruning policy — owner pins at implementation start.
- The linear default path remains the default; branching is an explicit action.

## Success criteria

- From a linear session, a branch created at turn N can be worked independently and both the branch and the main line remain addressable afterwards.
- Switching to a branch restores exactly the conversation state at its fork point (verifiable by transcript inspection).
- A substrate swap on a branched session preserves branch structure and per-branch state.

## Intent Approval

Status: colony intake at Thoor's direct directive (not colony-reviewed); GitHub issue approval pending (Thoor).
Approved by: pending (Thoor, on the GitHub issue).
Date: 2026-10-08

## Lineage

derived_from: soffio-fabric-spike (durable residency extension)
sources: huggingface/tau (session/tree.py), dissection doc above
