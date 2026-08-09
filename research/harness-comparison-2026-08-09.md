# Harness Comparison — 2026-08-09

Four parallel agent reviews of coding-agent-CLI relatives, run before
picking `pi` as Soffio's base. Kept verbatim (lightly trimmed) as
reference — this is the evidence behind "we picked `pi` after a 4-way
review" in `../intent.md`.

## claw-code (ultraworkers/claw-code) — skip, do not build on

Rust reimplementation of the Claude Code CLI harness. The README
itself says: **"Claw Code is not the serious production project
here"** — calls itself a "museum exhibit" run almost entirely by
autonomous agent harnesses, and tells readers who want to actually run
work to use LazyCodex or Gajae-Code instead. PHILOSOPHY.md: the
codebase is a "byproduct" — the real subject is the multi-agent
Discord-driven coordination system that generates it.

- Rust workspace, ~186K LOC across 11 crates. `tools` crate
  (11.8K LOC) is almost entirely one `lib.rs` (10,892 lines);
  `plugins` crate similarly concentrated (3,863-line `lib.rs`).
  `unsafe_code = "forbid"` + clippy `all=warn`, but only ~30 test files
  for 186K LOC.
- No plugin-authoring doc comparable to a real extension API;
  `docs/` is dominated by internal "g0xx-*-verification-map.md" audit
  artifacts, not developer guides.
- `cargo install claw-code` installs a deprecated stub binary (README
  warns about this itself).

**Ease to extend: 3/10.** The two crates most relevant to adding a
tool/skill are giant unstructured files, and the maintainers actively
point elsewhere. Reference/fossil only.

## pi (earendil-works) — chosen as Soffio's base

Pi Agent Harness — self-extensible coding-agent CLI + runtime.
npm-published (`@earendil-works/pi-coding-agent`), MIT, active
Discord/OSS community.

- TypeScript monorepo (npm workspaces), ~251K LOC hand-written.
  `packages/ai` (multi-provider LLM API), `packages/agent`
  (agent-core, tool calling), `packages/tui` (custom differential
  renderer, no external TUI framework), `packages/coding-agent` (the
  CLI). Biome for lint+format, `tsgo` for typecheck, Husky, exact-pinned
  deps + npm shrinkwrap.
- 438 `*.test.ts` files. `AGENTS.md` (11.6K) is unusually rigorous:
  dictates commit format, forbids `git add -A`/`--no-verify`, mandates
  full-file reads before wide edits, bans `any`, requires a
  faux-provider test harness (no real API calls in tests).
- **Extension system**
  (`packages/coding-agent/src/core/extensions/types.ts`, 1,727 lines /
  59.5K of typed API surface): lifecycle hooks, tool registration,
  commands, keybindings, UI dialogs/widgets/overlays, RPC.
  `docs/extensions.md` is 2,988 lines / 118K. `examples/extensions/`
  has 60+ working reference extensions, from `hello.ts` to
  `sandbox/`/`gondolin/` micro-VM isolation to `subagent/`.
- **Implements the open Agent Skills standard (agentskills.io) and
  loads Claude Code's and Codex's skill directories directly**
  (`~/.claude/skills`, `~/.codex/skills`) — real cross-harness
  interop, not a walled garden. This is the single deciding factor
  for choosing `pi`: Soffio inherits the ability to run skills built
  for other harnesses without translation.
- No built-in permission/sandboxing system — explicit design choice
  (`docs/containerization.md`); security is "your responsibility,"
  addressed via Gondolin micro-VM extension, Docker, or OpenShell.
  This is exactly the gap Soffio's self-responsibility gate has to
  fill.
- `tui-plan.md` (35.6K) at repo root shows the TUI layout system is
  mid-rewrite (stub-only interfaces) — some churn risk.

**Ease to extend: 8/10.**

## catcode (Catalyst Code, formerly Umans Harness) — strongest architecture

Self-hosted, provider-independent harness (terminal + browser). Daily
commits.

- Four components around one versioned stdio JSONL protocol
  (`protocol/protocol.schema.json`, v2): `core/` (Rust/tokio, 87K
  LOC), `tui/` (Go/Bubble Tea, 37.5K LOC), `sdk/` (TS, pi-compatible
  wrapper, 5.7K LOC — a real drop-in shim, not a rewrite), `web/`
  (Next.js/React, 30.6K LOC). New frontends just need a JSONL client.
- **Real sandboxing**: `core/src/sandbox/` `ExecutionBackend` trait,
  `HostExecutionBackend`/`MicrosandboxExecutionBackend` — bash tool
  can run in an optional microVM (separate kernel+FS root,
  `--no-network` opt), not a denylist tripwire. Path-confinement
  rejects `..`/absolute/symlink escapes.
- **Plugin system** (`core/src/plugins.rs`, 6.8K lines): hooks as
  subprocesses, JSON stdin/stdout, bounded I/O (1MB caps), timeouts,
  fail-safe. Repo-local plugins need explicit `--trust-project-plugins`
  — can't be silently enabled via checked-in config.
- 82/110 Rust files have tests; Go side has ~40 `_test.go` incl.
  golden/visual/e2e. But three Rust god-files (`main.rs` 5,780 lines,
  `provider.rs` 6,447, `plugins.rs` 6,762) are *growing*, not
  shrinking, relative to the repo's own stale `architecture-baseline.json`.
  Go mirrors this (`modal.go` 168KB, `handlers.go` 113KB).
- Docs include several huge raw-investigation-log files
  (WEB_FRONTEND_BUG_AUDIT.md 35.7K, SELF_LEARNING.md 45.3K) — good
  transparency, poor navigability.

**Ease to extend: 7/10** — low friction for hooks/skills (sandboxed
from core, has a dedicated `plugin-authoring` skill), high friction
for a core-native tool (touching multi-thousand-line god-files).

## oh-my-pi (can1357/oh-my-pi) — most features, real maintenance tax

Fork of `pi-mono` (mariozechner), maintained by Can Boluk. 40+ LLM
providers, 32 built-in tools, 14 LSP ops, 28 DAP/debugger ops.

- TypeScript/Bun dominant (~1.0M LOC), Rust in `crates/` (~137.7K
  LOC, but ~67.4K is vendored `brush-core`/`brush-builtins`, a forked
  bash shell — only ~70.3K first-party) compiled to a native N-API
  addon (`@oh-my-pi/pi-natives`) for perf-critical grep/glob/AST/
  shell/fs-walk. Python (`python/robomp`, ~36.6K LOC) is a fully
  separate consumer: a GitHub triage/fix bot driving `omp --mode rpc`
  over subprocess+JSON-RPC, not a first-class runtime participant.
- ~65 docs files, each pointing at exact implementation files —
  unusually agent-friendly documentation style. Layered extension
  surface: Skills, Custom Tools, Extensions (full lifecycle hooks),
  external Hooks, MCP integration, `docs/adding-a-provider.md`,
  `docs/porting-to-natives.md`.
- 1,536 `*.test.ts` files, strict TypeScript, aggressive Rust clippy
  lints. Panic-safety across the Rust↔TS FFI boundary is explicitly
  engineered (`crash_handler.rs`, unwind-catching before the napi
  C-unwind boundary).
- **`python/robomp`** — credential-isolated sidecar architecture ("holds
  the HMAC key, never the PAT"), per-issue git worktrees, session
  resumption. Worth studying independently of the Soffio decision.
- Weaknesses: `agent-session.ts` is 15,855 lines (god-file outlier);
  ~49% of Rust LOC is vendored shell needing manual fork-sync
  (`docs/porting-from-pi-mono.md` — hand-maintained "last sync commit"
  pointer, no automation); Rust/TS coupling is tight/fragile.

**Ease to extend: 7/10** — well-abstracted extension surface, docked
for native-binding fragility and the ongoing fork-sync tax.

## Bottom line

`pi` won on the one thing that matters most for a colony harness:
cross-harness skill compatibility, so Soffio doesn't lock the colony
into one vendor's skill format. `catcode`'s protocol-as-boundary and
real sandboxing are worth studying for Rialto's transport design and
the self-responsibility gate, respectively — borrow patterns, not
code, same rule as the letta-code note in `../intent.md`.
