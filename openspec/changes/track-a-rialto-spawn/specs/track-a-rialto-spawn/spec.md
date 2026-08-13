## ADDED Requirements

### Requirement: soffio requires an --agent flag to run
`soffio` SHALL require an `--agent <name>` CLI flag naming the
identity to run. Without it, `soffio` SHALL print help and exit
without doing any join/run work — this is the real CLI entry
condition, grounded in `pi`'s existing `parseArgs`/`printHelp`
(`packages/coding-agent/src/cli/args.ts`), extended with an `agent`
field on `Args` rather than a new flag-parsing layer.

#### Scenario: soffio invoked without --agent prints help and exits
- **WHEN** `soffio` is invoked with no `--agent` flag
- **THEN** help text is printed (via `pi`'s existing `printHelp`)
- **THEN** the process exits without attempting any join to a runtime
- **THEN** no partial state (join attempt, `pi` agent loop instance)
  is created

#### Scenario: soffio invoked with --agent proceeds to join that identity
- **WHEN** `soffio` is invoked with `--agent "<name>"`
- **THEN** `Args.agent` is set to `"<name>"`
- **THEN** the join-and-run path (below) begins for that identity

### Requirement: Rialto joins a named identity's own runtime
Rialto SHALL join the identity's own runtime (`coda-runtime` for the
MVP slice) via its existing `POST /v1/join` protocol to obtain that
identity's substrate — not construct a new runtime, and not import
ANIMA's `AgentSession` (ANIMA's own loop, for ANIMA's own agents,
unrelated to Soffio).

#### Scenario: Join a single identity by name
- **WHEN** Rialto is asked to join identity `"<name>"` (from
  `Args.agent`)
- **THEN** a `POST /v1/join` call is made to that identity's runtime
  endpoint with a valid `JoinCodaInput`
- **THEN** a `CodaJoinResponse` (`leaseId`, `capabilityToken`,
  `projection`, ...) is received without error

#### Scenario: Unknown identity name is rejected before any join attempt
- **WHEN** Rialto is asked to join an identity name with no known
  runtime endpoint
- **THEN** the join call fails before any HTTP request is attempted
- **THEN** no partial lease/session state is left registered

### Requirement: pi's own agent loop runs the identity's turn
Rialto SHALL feed the joined identity's substrate into `pi`'s own
agent loop (`packages/agent/src/agent.ts` + `agent-loop.ts`, used
unmodified) to run the actual chat/tool-execution turn — not a new
execution mechanism.

#### Scenario: A real turn runs using the joined substrate
- **GIVEN** a successful join for identity `"<name>"`
- **WHEN** `pi`'s agent loop is started with that identity's substrate
  as its system/context input
- **THEN** the loop runs a real turn (tool calls execute locally,
  inside `pi`'s own loop, not routed through the identity's runtime)

### Requirement: model override switches the backing provider without new plumbing
The provider/model backing a joined identity's turn SHALL be
swappable per run via `pi`'s own existing provider-override mechanism
— proving substrate-aware spawn needs no new API on either the Soffio
side or the ANIMA side.

#### Scenario: Same identity, two different models, same joined substrate
- **GIVEN** a joined identity has completed one turn with `pi`'s
  default model
- **WHEN** the same identity's next turn is run with `pi`'s own model
  override set to a different model
- **THEN** the second turn's response reflects the overridden model,
  not the default
- **THEN** the identity's own substrate/continuity state (obtained via
  the join) is unchanged by the override — only the model call changed

#### Scenario: Invalid model override surfaces a clear error, not a silent fallback
- **WHEN** `pi`'s model override is set to a model reference `pi`
  cannot resolve
- **THEN** the turn fails with an error naming the invalid reference
- **THEN** the turn does NOT silently fall back to the default model
