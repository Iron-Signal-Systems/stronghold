# Stronghold Go Project Boundaries

Stronghold is a single repository containing multiple cooperating products. Their implementations must remain separated even when they share the same repository and engineering standard.

Current reserved implementation boundaries:

```text
go/
├── access/   Stronghold Access control system
└── agent/    Stronghold Agent endpoint client / endpoint PEP
```

Stronghold FW implementation remains governed by the existing FW/capture architecture and parent `go/AGENTS.md`. As implementation grows, FW-specific code must not be moved into `go/access/` or `go/agent/` merely for convenience.

## Stronghold Access

```text
go/access/
```

Owns the Stronghold Access control-plane product: identity inputs, 802.1X/network-admission integration, AAA integration, posture and device-trust inputs, Access Sessions, authorization, revocation, Agent policy/session distribution, and the controlled Stronghold FW bolt-on interface.

See:

```text
docs/access/ARCHITECTURE.md
go/access/AGENTS.md
```

## Stronghold Agent

```text
go/agent/
```

Owns the endpoint application: Windows-first native endpoint integration, endpoint identity/enrichment, process-aware endpoint enforcement, Protected Endpoint transport, Agent health, and endpoint decision records.

See:

```text
docs/agent/ARCHITECTURE.md
go/agent/AGENTS.md
```

## Cross-Project Rule

Do not make the monorepo one program by importing another product's internal packages.

```text
Access internals != Agent internals
Agent internals  != FW internals
Access internals != FW internals
```

Products communicate through frozen, explicitly versioned contracts.

If shared protocol/schema code later becomes necessary, the protocol must be designed and versioned first. Do not create generic `common`, `shared`, `util`, `helpers`, or `framework` packages as a shortcut around product boundaries.

## Module Layout

The exact future `go.mod` / multi-module layout is intentionally not frozen by this file. Project separation is mandatory now; module mechanics should be selected when the first implementation phase requires them and should reinforce, not weaken, these boundaries.
