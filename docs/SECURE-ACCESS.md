# Stronghold Secure Access Protocol Architecture

## Status and Authority

This document preserves the **secure-transport and access-session architecture** for Stronghold.

Stronghold is one tightly coupled platform. Secure access is not a separate product family.

Authoritative component ownership is defined by:

```text
docs/PROJECT-BOUNDARIES.md
docs/access/ARCHITECTURE.md
docs/agent/ARCHITECTURE.md
```

This document governs the protocol split and cross-component secure-access invariants:

```text
ZERO-TRUST REMOTE ACCESS
    NIST SP 800-207 control-model direction
    Stronghold Access owns PE / PA authorization coordination
    Stronghold Agent is the endpoint PEP
    Stronghold FW is the independent network PEP
    WireGuard is the secure transport/data plane

SITE-TO-SITE / BRANCH OFFICE
    L2TPv3 tunnel
    IPsec protection
    site / tunnel identity
    Stronghold bridge / VLAN / routing integration
    normal Stronghold FW policy enforcement
```

**Implementation remains deferred and outside Phase 0.**

> **Path availability is not path permission.**

# Zero-Trust Remote Access

## Control and Data Plane

Stronghold follows the NIST SP 800-207 separation between authorization/control and enforcement.

```text
                    STRONGHOLD ACCESS
                 Policy Engine / Policy Admin
                           |
               GRANT / DENY / REVOKE
                           |
             +-------------+-------------+
             |                           |
             v                           v
      STRONGHOLD AGENT             STRONGHOLD FW
      endpoint PEP                 network PEP
      Windows / WFP                normal FW policy
             |                           ^
             |       WireGuard           |
             +---------------------------+
```

Stronghold Access coordinates authorization. Stronghold Agent and Stronghold FW enforce at different locations.

```text
Access GRANT
!=
Agent ALLOW
!=
FW ALLOW
```

A decision at one enforcement point never fabricates a decision at another.

## WireGuard Boundary

WireGuard is the core secure remote-access transport/data plane.

> **WireGuard proves and protects the encrypted peer-to-peer transport. Stronghold decides what that authenticated transport may access.**

Mandatory separations:

```text
WireGuard peer authenticated != user authenticated
user authenticated           != device authorized
device authorized             != resource authorized
tunnel established            != resource authorized
resource authorized           != connection succeeded
AllowedIPs configured         != Stronghold resource authorization
```

WireGuard peer state must not become the Stronghold authorization database.

## Stronghold Access Session

The conceptual authorization object is the **Stronghold Access Session**, not a permanently configured WireGuard peer entry.

Potential session facts include:

```text
Access Session ID
subject / user identity
device identity
device trust / certificate identity
MFA state
posture generation
application / process identity where established
Policy Generation
requested resource
protocol / service
grant scope
created / expiration time
authorization state
revocation reason
WireGuard transport/session reference where used
```

The session model is owned by `docs/access/ARCHITECTURE.md`.

## Revocation

Revocation is first-class.

Potential triggers include:

```text
account disabled
device certificate revoked
MFA/session expiration
posture failure
policy generation change
administrative revoke
Access Session expiration
network-admission change
approved Pathfinder risk/intelligence change
```

Conceptually:

```text
condition changes
      ↓
Stronghold Access reevaluates
      ↓
REVOKE / NEW AUTHORIZATION STATE
      ↓
Agent PEP update
      +
FW PEP update
      ↓
actual result independently recorded
```

```text
REVOKE issued
!=
revocation fully enforced
```

Stronghold must record the actual outcome at each enforcement point.

## Stronghold Agent

Stronghold Agent owns endpoint-side enforcement and protected transport behavior.

The first Windows direction is:

```text
Go service
    ↓
Windows Filtering Platform
    ↓
process/application-aware endpoint PEP
```

The Agent may stop unauthorized application/process connections before they enter WireGuard or the local network.

```text
endpoint ALLOW != FW ALLOW
endpoint DENY  != FW observed DENY
```

General deep-payload Layer-7 inspection is not an initial Agent requirement.

See `docs/agent/ARCHITECTURE.md`.

## Protected Endpoint

Selected high-value endpoints may later use a Protected Endpoint profile in which eligible traffic is forced through authenticated encrypted Stronghold transport toward the authorized Stronghold enforcement point.

A full-tunnel claim must account for IPv4, IPv6, DNS, local-subnet routing, secondary NICs, virtual adapters, sleep/resume, network transitions, local-link protocols, Agent failure, transport failure, and local-administrator tampering.

```text
protected payload unreadable
!=
traffic existence invisible
```

# Site-to-Site / Branch Office VPN

Site-to-site / branch-office tunneling is different from user/device zero-trust remote access.

Current protocol direction:

```text
L2TPv3 tunnel
      ↓
IPsec protection
      ↓
Stronghold site / tunnel identity
      ↓
bridge / VLAN / routing integration
      ↓
normal Stronghold FW policy
```

L2TPv3 provides the Layer-2 pseudowire/tunnel mechanism. IPsec protects that transport.

Mandatory separations:

```text
L2TPv3 tunnel established != site authorized
IPsec SA established      != traffic authorized
remote network reachable  != remote network permitted
tunnel healthy            != application connection succeeded
```

The tunnel provides a protected path. Stronghold policy provides permission.

Future site-tunnel work must define site/tunnel identity, endpoint authentication, IKE/IPsec profile, VLAN/bridge/routing behavior, MTU/fragmentation, multi-WAN interaction, HA/failover/rekey behavior, journaling, health, and interoperability.

# Pathfinder Relationship

Pathfinder is a separate Iron Signal Systems threat-intelligence system, not part of the secure transport and not an authorization authority by itself.

Pathfinder may provide risk/intelligence context to Stronghold Access and may enrich traffic/IDS/Hunter history.

```text
Pathfinder risk signal
!=
automatic Access REVOKE

Pathfinder match
!=
WireGuard/session authorization
```

Stronghold Access determines the actual authorization result under a known policy generation.

Stronghold Agent normally receives the resulting authorized policy from Access rather than querying Pathfinder directly.

See `docs/PATHFINDER-INTEGRATION.md`.

# Observation and Journaling

Secure-access traffic remains subject to Stronghold's physical-interface observation model where presented to configured supported capture paths.

Stronghold preserves the difference between:

```text
encrypted WireGuard outer traffic
inner traffic after tunnel termination
L2TPv3/IPsec outer traffic
inner site traffic after decapsulation
Access authorization history
Agent endpoint decisions
FW traffic decisions
```

Exact capture representation across tunnel termination/decapsulation must be frozen before implementation so Stronghold does not double-count or misrepresent physical observation versus logical processing.

Relevant journal domains include Administrative, System/Health, Traffic Decision, Trust/Identity, and Time/Clock according to the event.

Net-Hunter may correlate Access Sessions, Agent decisions, FW decisions, tunnel state, Pathfinder intelligence, and authoritative PCAP without rewriting source history.

# Failure and Health

Secure-access health is domain-specific.

Potential remote-access states include:

```text
PE / PA availability
Agent PEP health
FW PEP health
identity provider
MFA dependency
posture dependency
WireGuard transport
Access Session state
revocation processing
Pathfinder intelligence freshness where policy uses it
```

Potential site-tunnel states include:

```text
L2TPv3 tunnel state
IPsec SA / rekey state
peer authentication
endpoint/WAN reachability
bridge/VLAN/route integration
packet/drop/error state
```

A healthy secure transport never proves authorization or application success.

No transport failure silently creates plaintext fallback.

```text
WireGuard unavailable != permission to fall back to plaintext remote access
IPsec failed          != permission to run L2TPv3 unprotected
identity unavailable  != user automatically authorized
posture unavailable   != posture passed
Pathfinder unavailable != observable trusted
```

# Qualification Requirements

Before zero-trust remote access implementation, freeze and validate:

```text
PE / PA / PEP responsibility boundaries
Stronghold Access Session schema/lifecycle
user/device identity
MFA / posture
Agent enrollment/update
WireGuard key/session lifecycle
AllowedIPs vs Stronghold authorization
resource authorization
grant/deny/revoke
control-plane outage behavior
policy leases / holdover
Agent/FW dual-PEP semantics
capture/journal representation
Hunter correlation
Pathfinder-influenced policy provenance
management/API/RBAC
health/alerting
performance/scaling
```

Before site-to-site / branch-office implementation, freeze and validate:

```text
L2TPv3 profile
IPsec/IKE profile
peer/site authentication
site/tunnel identity
bridge/VLAN/zone integration
routing interaction
MTU/fragmentation
multi-WAN interaction
HA/failover/rekey
packet-observation representation
journal events
management/API/RBAC
health/alerting
interoperability matrix
performance/PPS/throughput
```

# Scope

Secure-access implementation remains deferred.

Nothing in this document pulls Stronghold Access, Agent, WireGuard, L2TPv3/IPsec, Pathfinder integration, or secure-access HA into Phase 0.
