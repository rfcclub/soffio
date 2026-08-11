# AI Agent Digest — 2026-08-10 notes

thoor forwarded a 3-item digest. Two of the three are direct prior art
for Soffio; recorded here with the primary sources actually read
(not just the digest summary), per this repo's rule of citing real
sources.

## AgentRadio — validates the core coordination thesis, real mechanism to study for Rialto

Primary source: [arXiv 2607.28430](https://arxiv.org/html/2607.28430)
(paper), [github.com/Coral-Protocol/AgentRadio](https://github.com/Coral-Protocol/AgentRadio)
(code) — not just the VentureBeat summary.

**The problem it solves**: existing multi-agent systems only let agents
exchange information at phase boundaries (staged handoffs, synchronized
rounds) — "communication and work remain mutually exclusive." A
discovery one agent makes mid-task can't reach a teammate until the
next boundary, even though code-comprehension subtasks are
interdependent (one agent's finding can invalidate what a teammate is
doing right now).

**The mechanism**: three primitives — `create_thread(name,
participants)`, `send_message(thread, content, mentions)` (returns
immediately, non-blocking), `wait_for_mention(timeout)` (blocks until
mentioned, returns the message + a full thread snapshot). The one
finding that matters: **where `wait_for_mention` runs is the whole
result.** Run in the foreground, it's a blocking receive — the agent
stops working to listen. Run as a background task of the harness, it
becomes "passive awareness" — the agent keeps working, mentions surface
at the next step boundary, "much like a radio reaches a driver whose
hands never leave the wheel." Implementation-wise this is genuinely
cheap: the message server is a standalone process, each agent reaches
it through thin shell scripts, and the harness itself needs no
modification beyond running a background shell command — capability
mainstream harnesses (including `pi`) already have.

**The result**: under a 5-phase division-of-labor + negotiation
protocol, 4 Claude Code (Opus 4.6) agents wired with AgentRadio resolve
62.1% of SWE-Atlas QnA tasks vs. 32.3% for one agent alone and 57.2%
for a single agent on the stronger Opus 4.8 — beating a full model
generation upgrade with a coordination-layer change alone. Ablation
isolates the passive-vs-blocking difference specifically: passive
awareness alone adds +10.5 points (Opus 4.6) / +11.3 (DeepSeek V4 Pro)
over the same protocol run with blocking receive — same primitives,
same protocol, only the listen-in-foreground-vs-background bit
changes. Also beats compute-matched best-of-6 sampling (37.9% vs.
62.1%) — "structure over compute."

**Why this matters for Soffio**: this is direct, measured evidence for
the thesis GardenHub/Rialto already assumes (colony coordination beats
a single strong model) — worth having as a citable result, not just an
intuition. More concretely: GardenHub's `TickEngine` is currently
poll/tick-based (per `src/garden/tick-engine.ts`); AgentRadio's
background-listener pattern is a candidate upgrade for how spawned
Soffio agents get mid-task mentions from GardenHub or from each other
without blocking on a poll cycle. Not adopting the code (it's a
research implementation, MIT-licensed but scoped to their 5-phase
protocol) — the primitive worth stealing is the *background
wait_for_mention* idea itself, layered onto Rialto's own transport.

## Docker Sandboxes — confirms the microVM sandboxing choice, doesn't change the design

[docker.com/products/docker-sandboxes](https://www.docker.com/products/docker-sandboxes/).
Disposable microVM-isolated environments for coding agents (Claude
Code, Gemini CLI, Copilot CLI, Codex, OpenCode, Kiro out of the box),
network/filesystem controls, agents can run nested Docker containers
inside. This is the same microVM-over-denylist approach already noted
as catcode's design in
[`harness-comparison-2026-08-09.md`](./harness-comparison-2026-08-09.md)
(their `MicrosandboxExecutionBackend`). Docker productizing the same
pattern generally is a signal the approach is right, not new
information that changes Soffio's self-responsibility-gate design —
recorded for completeness, no action needed. Worth a look later as a
possible *implementation* of the gate's sandbox layer instead of
hand-rolling one, once Soffio needs it — Docker Sandboxes' agent list
doesn't include `pi`/oh-my-pi/catcode/Soffio by name, but the page
says custom harnesses are supported.

## Stanford "Virtual Biotech" (37,000 agents) — scale data point, not a mechanism

[VentureBeat coverage](https://venturebeat.com/orchestration/stanford-is-running-37-000-ai-agents-as-a-virtual-biotech-and-one-of-its-drug-designs-got-independently-confirmed-by-merck) —
not deep-dived, no primary source read yet. Notable as an existence
proof that colony-scale multi-agent orchestration (structured like a
company — a CSO-equivalent agent managing specialized divisions) works
at far larger scale than Soffio's 16-agent target, and that Merck
independently confirmed a real-world result (a drug design) it
produced. Doesn't currently change any Soffio design decision — flagged
in case the org-structure pattern (one coordinating agent managing
specialized division-agents) becomes relevant once Soffio's `/goal`
equivalent (see
[`feature-adoption-2026-08-09.md`](./feature-adoption-2026-08-09.md))
gets designed in earnest.
