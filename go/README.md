# Stronghold Go Component Boundaries

Stronghold is one tightly coupled security platform in a single repository. Its components have explicit implementation boundaries so cooperation does not become a monolithic program or blurred authority model.

Current reserved implementation boundaries:

```text
go/
├── access/   Stronghold Access infrastructure component
└── agent/    Stronghold Agent endpoint component / endpoint PEP
```

Stronghold FW implementation remains governed by the existing FW/capture architecture and parent `go/AGENTS.md`. As implementation grows, FW-specific code must not be moved into `go/access/` or `go/agent/` merely for convenience.

## Stronghold Access

```text
go/access/
```

Owns the Stronghold Access control-plane implementation: identity inputs, 802.1X/network-admission integration, AAA integration, posture and device-trust inputs, Access Sessions, authorization, revocation, Agent policy/session distribution, controlled Stronghold FW integration, and later approved Pathfinder intelligence/risk inputs.

Stronghold Access is the third Stronghold infrastructure node when deployed and is intended to run on a supported customer VM or supported bare-metal server.

See:

```text
docs/PROJECT-BOUNDARIES.md
docs/access/ARCHITECTURE.md
go/access/AGENTS.md
```

## Stronghold Agent

```text
go/agent/
```

Owns the endpoint implementation: Windows-first native endpoint integration, endpoint identity/enrichment, process-aware WFP enforcement, Protected Endpoint transport, Agent health, and endpoint decision records.

Agent is an endpoint component of the Stronghold platform, not a separate unrelated product.

See:

```text
docs/agent/ARCHITECTURE.md
go/agent/AGENTS.md
```

## Cross-Component Rule

Do not make the monorepo one process by importing another component's internal packages.

```text
Access internals != Agent internals
Agent internals  != FW internals
Access internals != FW internals
```

Stronghold components communicate through frozen, explicitly versioned contracts.

Pathfinder is a separate ISS system and does not become a Go subtree merely because Stronghold integrates with it. The Stronghold↔Pathfinder boundary is governed by `docs/PATHFINDER-INTEGRATION.md`.

If shared Stronghold protocol/schema code later becomes necessary, the protocol must be designed and versioned first. Do not create generic `common`, `shared`, `util`, `helpers`, or `framework` packages as a shortcut around component boundaries.

## Module Layout

The exact future `go.mod` / multi-module layout is intentionally not frozen by this file. Component separation is mandatory now; module mechanics should be selected when the first implementation phase requires them and should reinforce, not weaken, these boundaries.
