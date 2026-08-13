## ADDED Requirements

### Requirement: Rialto spawns a colony identity as a live AgentSession
Rialto SHALL construct one `LoadedAgent` and one `Provider` for a
named colony identity and run an `AgentSession` against them, reusing
`agent-runtime`'s existing loader/registry/session code without
modification.

#### Scenario: Spawn a single identity by name
- **WHEN** Rialto is asked to spawn identity `"<name>"` from the
  colony roster
- **THEN** a `LoadedAgent` is constructed via the existing agent
  loader for that identity's room
- **THEN** a `Provider` is resolved via `ProviderRegistry.resolve()`
  for that identity's configured default model
- **THEN** an `AgentSession` is constructed with that store and
  provider without error

#### Scenario: Unknown identity name is rejected before any spawn work happens
- **WHEN** Rialto is asked to spawn an identity name not present in
  the colony roster
- **THEN** the spawn call throws before constructing a `LoadedAgent`
  or `Provider`
- **THEN** no partial `AgentSession` is left constructed

### Requirement: modelOverride switches the backing provider without new plumbing
A spawned identity's provider/model SHALL be swappable per run via
`RunAgentSessionOptions.modelOverride`, proving substrate-aware spawn
needs no new API beyond what `agent-runtime` already exposes.

#### Scenario: Same identity, two different models, same session flow
- **GIVEN** a spawned identity's `AgentSession` has completed one run
  with its default model
- **WHEN** the same identity is run again with
  `modelOverride` set to a different `model@provider` ref
- **THEN** the second run's `ProviderResponse` usage/metadata reflects
  the overridden provider, not the default
- **THEN** the identity's own state (continuity DB path, memory
  store) is unchanged by the override — only the provider call
  changed

#### Scenario: Invalid modelOverride ref surfaces a clear error, not a silent fallback
- **WHEN** `modelOverride` is set to a `model@provider` ref that
  `ProviderRegistry.resolve()` cannot resolve
- **THEN** the run fails with an error naming the invalid ref
- **THEN** the run does NOT silently fall back to the identity's
  default model
