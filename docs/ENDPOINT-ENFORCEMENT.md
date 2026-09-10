# Stronghold Endpoint Enforcement Supporting Architecture

## Status and Authority

This document preserves the endpoint-enforcement design principles that led to the current Stronghold Agent architecture.

Authoritative component ownership and implementation rules are now defined by:

```text
docs/PROJECT-BOUNDARIES.md
docs/access/ARCHITECTURE.md
docs/agent/ARCHITECTURE.md
go/access/AGENTS.md
go/agent/AGENTS.md
```

Stronghold is one tightly coupled platform. Endpoint enforcement is implemented by the **Stronghold Agent endpoint component** and coordinated by the **Stronghold Access infrastructure component**.

**Endpoint-enforcement implementation remains deferred and outside Phase 0.**

The governing idea remains:

> **Stronghold may enforce policy at the endpoint before unauthorized traffic reaches the network, while Stronghold FW remains the independent network enforcement authority.**

# Component Relationship

```text
Stronghold FW / Network Admission
    establish network-side context
        |
        v
Stronghold Access
    identity / posture / policy / Access Session
    optional Pathfinder intelligence/risk input
        |
        | signed monotonic Agent policy generation
        v
Stronghold Agent
    Windows service / Go
        |
        v
Windows Filtering Platform
        |
        v
endpoint PEP
        |
   +----+----+
   |         |
 DENY      ALLOW
   |         |
local        v
record    network / WireGuard
             |
             v
        Stronghold FW
             |
             v
       independent FW policy
```

This is a dual-PEP model, not a way to move FW authority onto the endpoint.

# Governing Authorization Boundaries

```text
endpoint ALLOW          != FW ALLOW
endpoint DENY           != FW observed DENY
endpoint policy active  != FW policy active
endpoint PEP healthy    != FW PEP healthy
application identified  != application trusted
device on VLAN          != device authorized for VLAN/zone
Pathfinder match        != endpoint DENY authority
```

A connection that survives endpoint enforcement still requires normal Stronghold FW authorization when it reaches the FW.

If the Agent denies a connection before transmission, Stronghold records an endpoint decision and does not invent a FW packet observation.

# Initial Windows Direction

The initial Agent platform direction is Windows.

```text
Stronghold Agent
    Windows service
    Go
        ↓
Windows-native APIs
        ↓
Windows Filtering Platform
        ↓
process/application-aware endpoint enforcement
```

The first implementation should use Windows-native enforcement capabilities rather than build a competing user-space packet stack or repeatedly parse shell output.

Potential local authorization facts include:

```text
user / subject identity
device identity
application / process identity
process path / publisher state where established
destination / resource
service / port
protocol
direction
Stronghold Access Session
Policy Generation
network context
posture context
```

# Layer-7 Boundary

The initial Agent may make **application/process-aware** decisions without claiming general deep payload inspection.

```text
originating process identified
!=
payload decoded

application-aware connection policy
!=
deep Layer-7 payload inspection
```

Any future local proxy, stream-inspection design, or kernel-mode WFP callout driver requires separate approval and qualification for protocol coverage, TLS handling, driver signing, kernel attack surface, crash/BSOD risk, Windows compatibility, performance, privacy, logging/retention, diagnostics, and fail-open/fail-closed behavior.

# Network Context

Stronghold Agent may report local interface/address observations, but it cannot self-assert authoritative network trust.

Network-side VLAN/zone/attachment facts come from Stronghold FW or qualified network-admission sources and are correlated by Stronghold Access.

```text
Agent reports VLAN / zone
!=
network-side VLAN / zone established
```

A network-context change may cause Access to generate a new signed Agent policy generation.

The FW does not directly micromanage individual WFP filters with ad hoc per-packet commands.

# Policy Generation and Holdover

Endpoint policy should be signed, versioned, monotonic, and bounded by explicit lease/holdover semantics.

Conceptual lifecycle:

```text
RECEIVED
    ↓
VERIFIED
    ↓
VALIDATED
    ↓
WFP PROGRAMMED
    ↓
ACTIVATED
```

Operational states may include:

```text
POLICY_CURRENT
POLICY_HOLDOVER
POLICY_EXPIRED
CONTROL_UNAVAILABLE
ENFORCEMENT_DEGRADED
ENFORCEMENT_FAILED
```

Stronghold must not globally assume either:

```text
control unavailable -> allow everything
```

or:

```text
control unavailable -> brick all endpoint networking
```

Exact behavior is profile- and policy-specific.

# Protected Endpoint

Selected high-value endpoints may later use a Protected Endpoint profile in which eligible traffic is forced through an authenticated encrypted Stronghold transport toward the authorized Stronghold enforcement point.

Potential use cases include privileged administration workstations, finance/CJIS endpoints, domain infrastructure, backup infrastructure, critical servers, and other explicitly designated systems.

A full-tunnel claim must account for bypass/leak surfaces including:

```text
IPv4 / IPv6
DNS
local-subnet routes
secondary NICs
Wi-Fi + Ethernet concurrency
USB NICs
virtual adapters / Hyper-V
other VPN software
boot/startup
sleep/resume
network transitions
multicast/broadcast
DHCP/DHCPv6
ARP/NDP
Agent failure
transport failure
local administrator tampering
```

```text
protected payload unreadable
!=
traffic existence invisible
```

# WireGuard Relationship

WireGuard is the Stronghold zero-trust remote-access transport direction.

```text
WireGuard peer authenticated != user authenticated
tunnel established           != resource authorized
endpoint process authorized  != FW resource authorized
```

WireGuard provides protected transport. Stronghold Access and Stronghold FW determine authorization.

# Pathfinder Relationship

Pathfinder is a separate ISS threat-intelligence system.

Stronghold Agent should normally not query Pathfinder directly.

Preferred path:

```text
Pathfinder
    ↓
Stronghold Access policy evaluation
    ↓
signed Stronghold Agent policy
    ↓
Stronghold Agent enforcement
```

The Agent may enforce policy influenced by Pathfinder context, but a Pathfinder match is not itself an endpoint command.

```text
Pathfinder match       != Agent DENY authority
Pathfinder risk signal != endpoint compromise proven
```

See `docs/PATHFINDER-INTEGRATION.md`.

# Endpoint Decision Records

Endpoint decisions should be attributable and correlatable.

Potential record fields include:

```text
Endpoint Decision ID
Endpoint / Device ID
user / subject identity
application / process identity
source network context
destination / resource
service / protocol
Access Session ID
Policy Generation
decision
reason
result
UTC time
endpoint clock-confidence state
enforcement health state
```

Potential results include:

```text
ALLOW
DENY
REVOKED
NOT_PERFORMED
POLICY_UNAVAILABLE
POLICY_EXPIRED
ENFORCEMENT_FAILED
TRANSPORT_UNAVAILABLE
```

Endpoint decisions remain separate from FW Traffic Decision Journal facts.

# Hunter Correlation

Net-Hunter may correlate:

```text
Endpoint Decision ID
Endpoint / Device ID
user
application / process
Network Access Session
Stronghold Access Session
Policy Generation
network context
FW decision
WireGuard transport
Pathfinder Record / interpretation reference where policy used it
related packet segments
```

Correlation is derived interpretation. It does not convert an endpoint record into a FW source journal entry or physical-interface observation.

# Local Administrator Boundary

Stronghold must remain truthful about the Windows local-administrator/SYSTEM-equivalent threat boundary.

A sufficiently privileged local attacker may be able to tamper with services, filters, certificates, routes, drivers, boot state, or other endpoint controls.

Signing, protected identity material, anti-downgrade controls, health validation, and tamper detection may reduce risk but do not justify claiming immunity from a fully privileged compromised operating system.

# Qualification Requirements

Before production Agent endpoint enforcement is promoted into an implementation phase, freeze and validate:

```text
supported Windows versions / editions
Go service lifecycle / privileges
installer / uninstall / signing
update / rollback / anti-downgrade
endpoint enrollment / identity
certificate / key protection
Stronghold Access control protocol
policy bundle schema / signing
policy generation / lease / holdover
network-context source/generation
WFP integration / layer selection
filter ownership / cleanup
application / process identity semantics
user / device identity semantics
posture scope / freshness
Network Access Session correlation
Protected Endpoint transport
full-tunnel bypass/leak matrix
WireGuard integration
Agent / FW dual-PEP semantics
Pathfinder-influenced policy provenance
endpoint decision record integrity / transport
local-admin tamper boundary
performance / latency / resource use
sleep / resume / network transitions
multi-NIC behavior
other-VPN coexistence
failure / recovery
Hunter correlation
health / observability
```

# Scope

This is supporting architecture only.

The authoritative current Agent architecture is `docs/agent/ARCHITECTURE.md`. No Agent/WFP/deep-L7/Pathfinder implementation is pulled into Phase 0.
