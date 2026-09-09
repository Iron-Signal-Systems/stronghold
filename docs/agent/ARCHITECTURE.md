# Stronghold Agent Architecture

## Purpose

**Stronghold Agent** is the endpoint application for Stronghold-managed endpoint identity, enrichment, local enforcement, protected transport, and Stronghold Access participation.

Stronghold Agent is a separate implementation project from Stronghold Access and Stronghold FW.

Stronghold Agent is not an EDR, not a replacement enterprise identity provider, not an independent Stronghold Policy Engine, and not a hidden endpoint firewall product with its own unrelated policy model.

Its governing role is:

> **Establish endpoint-origin facts, enforce authorized policy locally, protect selected traffic before it crosses an untrusted network, and report what the endpoint actually decided and performed.**

## Product Boundary

```text
Stronghold Access
    control framework / policy authority

Stronghold Agent
    endpoint application / endpoint PEP

Stronghold FW
    independent network PEP
```

Implementation belongs under:

```text
go/agent/
```

Stronghold Agent must not import Stronghold Access internal implementation packages. Interaction occurs through explicitly versioned control, policy, identity, and session contracts.

## Initial Windows Direction

Windows is the first endpoint platform direction.

Conceptually:

```text
Stronghold Agent
    Windows service
    Go
        |
        v
Windows-native endpoint integration
        |
        v
Windows Filtering Platform
        |
        v
endpoint Policy Enforcement Point
```

Windows-owned behavior should use appropriate native Windows APIs and facilities rather than depend on repeated parsing of shell-command output.

The initial enforcement direction is process/application-aware connection authorization through Windows Filtering Platform before an unauthorized connection needs to traverse the LAN, WireGuard transport, or Stronghold FW.

The exact WFP layers, filter ownership model, service privileges, signing model, installation model, certificate/key storage, update mechanism, and supported Windows versions remain to be frozen before implementation.

## Endpoint Facts

Potential endpoint-origin facts include:

```text
user / subject identity
device identity
device certificate identity
application / process identity
process path
publisher / signature state where established
process hash where later justified
destination IP / resource
destination port / service
protocol
direction
network context
Stronghold Access Session
Policy Generation
posture state / generation
authorization state
```

Stronghold must preserve the source and confidence of each fact rather than merge all endpoint observations into one generic `trusted` state.

## Application / Process Identity

Endpoint process identity is different from traffic-derived application classification.

```text
Windows originating process
!=
network application interpretation
```

Example:

```text
User:        DOMAIN\John
Device:      LAPTOP-42
Application: mstsc.exe
Resource:    SERVER-17
Service:     TCP/3389
Decision:    ALLOW
```

while:

```text
User:        DOMAIN\John
Device:      LAPTOP-42
Application: powershell.exe
Resource:    SERVER-17
Service:     TCP/3389
Decision:    DENY
```

Same user, device, destination, and service may result in different endpoint decisions because the originating application is part of the authorization context.

Application identified does not mean application trusted.

## Local Endpoint PEP

Stronghold Agent may enforce policy locally before unauthorized traffic reaches the network.

```text
process attempts connection
        |
        v
Stronghold Agent / WFP PEP
        |
   +----+----+
   |         |
 DENY      ALLOW
   |         |
 local       v
 result   network / tunnel path
```

The purpose is not merely duplicate filtering. Local enforcement can reject clearly unauthorized application/process connections before Stronghold FW must spend resources processing them, while preserving the FW as an independent network authority for traffic that does leave the endpoint.

A locally denied connection must be recorded as an endpoint fact.

```text
endpoint denied before transmission
!=
FW denied packet
```

Stronghold must not fabricate a Stronghold FW observation when no packet was presented to the FW.

## Layer-7 Boundary

The initial Agent design may use **application/process identity** as an authorization fact without claiming that the endpoint is performing general deep packet inspection.

This distinction is mandatory:

```text
originating process identified
!=
payload decoded

application-aware connection policy
!=
deep Layer-7 payload inspection
```

The first Windows implementation direction is deliberately limited to native endpoint connection/application enforcement where practical through WFP and related Windows APIs.

Any future payload-aware capability that requires a local proxy, stream inspection, or kernel-mode WFP callout driver is a separate implementation and qualification decision. It must not be introduced merely to call the Agent a Layer-7 product.

A future deep-L7 design would require explicit review of protocol coverage, TLS handling, driver signing, kernel attack surface, crash/BSOD risk, upgrade compatibility, performance, privacy, logging/retention, and fail-open/fail-closed behavior.

## Stronghold Access Relationship

Stronghold Access is the control framework and policy authority for Agent-managed access behavior.

Conceptually:

```text
Stronghold Access
       |
       | authenticated / versioned policy
       | Access Session state
       | posture / revocation state
       v
Stronghold Agent
       |
       v
endpoint PEP
```

Potential policy inputs include:

```text
Endpoint / Device ID
Policy Generation
valid-from time
expiration / lease time
network context
resource/application/service authorization
Stronghold Access Session state
issuer identity
integrity/signature metadata
```

Policy received is not policy activated. Policy activated is not proof that WFP enforcement is healthy.

```text
policy received       != policy verified
policy verified       != policy activated
policy activated      != enforcement verified
Agent service running != endpoint PEP healthy
```

## Network Context from Stronghold Access / FW

The Agent may consume signed policy that reflects network-side context established by Stronghold Access, Stronghold FW, or qualified network-admission sources.

A preferred integrated flow is:

```text
Stronghold FW / network admission
    establishes VLAN / zone / attachment facts
        |
        v
Stronghold Access
    correlates endpoint identity + network context
    evaluates applicable endpoint policy
        |
        | signed monotonic policy generation
        v
Stronghold Agent
    verifies / validates / programs WFP
        |
        v
endpoint PEP
```

The Agent may also observe local interface/address information, but it must not use self-reported network state to promote itself into a trusted VLAN/zone or broader authorization scope.

```text
Agent reports VLAN / zone
!=
network-side VLAN / zone established

network context changed
!=
new endpoint policy activated
```

Network-context changes should cause policy reevaluation through Stronghold Access rather than trigger ad hoc per-packet commands from the FW. Policy distribution should use signed/versioned generations with explicit lease/holdover behavior.

## Operating Modes

Stronghold Agent should support explicit operating modes rather than silently changing network behavior.

### Managed Endpoint

```text
endpoint identity / enrichment
authorized local enforcement
normal network path where policy allows
Stronghold Access participation
```

The Agent may provide identity, posture, process-aware authorization, and local endpoint PEP behavior without forcing all traffic through a Stronghold tunnel.

### Protected Endpoint

Protected Endpoint is intended for selected high-value systems or departments where eligible traffic must not traverse the local network as plaintext application traffic.

Potential use cases include:

```text
privileged administration workstations
finance systems
CJIS workstations
executive systems
domain infrastructure
backup infrastructure
critical servers
selected department workstations
other explicitly designated high-value endpoints
```

Conceptually:

```text
HIGH-VALUE ENDPOINT
        |
Stronghold Agent / WFP PEP
        |
        | authenticated encrypted transport
        v
Stronghold FW / authorized Stronghold enforcement point
        |
        v
permitted resource
```

A passive observer on the local LAN may still observe that encrypted traffic exists, along with physical/network metadata such as MAC addressing, attachment addressing, packet timing, packet sizes, and volume where visible.

Stronghold must not claim that the endpoint becomes physically invisible.

```text
protected payload unreadable
!=
traffic existence invisible

protected payload unreadable
!=
transport cannot be disrupted
```

### Privileged Endpoint

A later stricter policy profile may combine Protected Endpoint transport with narrower application/resource authorization, stricter posture requirements, shorter authorization leases, and stronger authentication requirements.

This is a policy/profile distinction, not a separate hidden networking stack.

## Protected Endpoint Full-Tunnel Contract

If Stronghold describes a mode as full tunnel, the contract must be explicit and qualification-driven.

Potential bypass/leak surfaces that must be addressed include:

```text
IPv4
IPv6
DNS
local-subnet routes
secondary NICs
Wi-Fi + Ethernet concurrency
USB NICs
virtual adapters
Hyper-V networking
other VPN software
boot/startup before Agent readiness
sleep / resume
network transitions
route-table changes
multicast / broadcast behavior
DHCP / DHCPv6
ARP / NDP
local discovery protocols
Agent failure
transport failure
local administrator tampering
```

Some local-link traffic is inherently required for attachment and transport establishment. Stronghold must define exactly what remains outside the protected tunnel and why.

A full-tunnel claim must not be based only on installing a default route.

## WireGuard / Secure Transport

WireGuard is the current Stronghold zero-trust transport direction.

WireGuard establishes authenticated encrypted transport. It does not itself grant resource authorization.

```text
WireGuard peer authenticated
!=
user authenticated

tunnel established
!=
resource authorized

endpoint process authorized
!=
FW resource authorized
```

Protected Endpoint transport must preserve these boundaries.

Exact endpoint key lifecycle, peer provisioning, AllowedIPs behavior, tunnel addresses, rekey/rotation behavior, local secret protection, transport keepalive, revocation, and recovery remain to be frozen.

## Stronghold FW Relationship

When integrated with Stronghold FW, the FW remains an independent network PEP.

```text
Endpoint Agent ALLOW
        |
        v
protected / normal network path
        |
        v
Stronghold FW
        |
        v
independent FW authorization
```

An endpoint ALLOW never grants network permission that Stronghold FW would otherwise deny.

Stronghold FW may correlate authenticated Access Session and Agent-established facts with the network traffic actually presented to it.

Where the FW provides network-side context to Access, that context may influence the next signed Agent policy generation. The FW should not directly micromanage individual endpoint WFP filters outside the versioned Access/Agent contract.

## Network Admission / 802.1X Relationship

Stronghold Agent does not replace the operating system's appropriate 802.1X supplicant or enterprise network-admission system merely to centralize everything inside the Agent.

Stronghold Access owns the control/framework relationship with 802.1X and AAA.

The Agent may contribute endpoint/device identity or posture facts used by Stronghold Access, but:

```text
Agent says trusted network
!=
network admission established

Agent reports VLAN / zone
!=
network-side VLAN / zone established
```

See `docs/access/ARCHITECTURE.md`.

## Posture

Posture may become an Access authorization input, but Stronghold must define which posture facts are collected, where they originate, how fresh they are, and what happens when they cannot be established.

Potential posture inputs may include, subject to later approval and qualification:

```text
OS / supported version
patch state
local firewall state
disk-encryption state
approved security-service state
certificate / device identity state
Stronghold Agent health
selected configuration state
compromise / risk signal from an approved external source
```

Stronghold Agent must not become a general-purpose endpoint telemetry collector simply because Windows exposes additional data.

## Endpoint Decision Records

Endpoint decisions should be attributable and correlatable.

Potential record fields include:

```text
Endpoint Decision ID
Endpoint / Device ID
user / subject identity
application / process identity
process path / signature identity where established
source network context
network-context generation/source
destination / resource
service / protocol
Access Session ID
Policy Generation
decision
reason
result
UTC time
endpoint clock-confidence state
control-plane policy source
enforcement health state
```

Potential states/results include:

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

Exact schema remains future work.

## Local Administrator Boundary

Stronghold must remain truthful about the Windows local-administrator/SYSTEM-equivalent threat boundary.

A sufficiently privileged local attacker may be able to tamper with services, filters, certificates, routes, drivers, boot state, or other endpoint security controls.

The design should use appropriate signing, protected identity material, anti-downgrade controls, health validation, and tamper detection where practical, but it must not claim impossible immunity from a fully privileged compromised operating system.

## Failure and Holdover

Stronghold Agent must not silently disable security controls when Stronghold Access or Stronghold FW is unavailable.

Potential policy states include:

```text
POLICY_CURRENT
POLICY_HOLDOVER
POLICY_EXPIRED
CONTROL_UNAVAILABLE
ENFORCEMENT_DEGRADED
ENFORCEMENT_FAILED
TRANSPORT_DEGRADED
TRANSPORT_FAILED
```

Exact fail-open/fail-closed behavior must be explicit and scoped by profile/use case.

For Protected Endpoint or Privileged Endpoint profiles, policy may intentionally be substantially stricter than ordinary Managed Endpoint operation.

Stronghold must not globally assume either:

```text
control unavailable -> allow everything
```

or:

```text
control unavailable -> brick all endpoint networking
```

## Hunter Correlation

Net-Hunter may later correlate:

```text
Endpoint Decision ID
Endpoint / Device ID
user
application / process
Network Access Session
Stronghold Access Session
Policy Generation
network-context generation/source
resource
service
endpoint decision
FW decision
WireGuard transport
related packet segments
```

Correlation is derived interpretation. It does not convert an endpoint record into a Stronghold FW source journal entry or physical-interface observation.

## Future Platforms

Windows is the initial platform direction.

Future Linux or macOS support, if approved, must be implemented according to the native enforcement/security facilities of those operating systems rather than forcing Windows-specific abstractions onto them.

Platform behavior must remain truthful even when capabilities differ.

## Implementation Boundary

Stronghold Agent implementation belongs under:

```text
go/agent/
```

Stronghold Access implementation belongs under:

```text
go/access/
```

Do not create direct imports between one project's internal packages and the other project's internal packages.

If shared protocol/schema code later becomes necessary, freeze the protocol first and give the shared contract explicit ownership/versioning. Do not create generic `common`, `util`, `framework`, or `shared` packages as a shortcut around project boundaries.

## Qualification Requirements

Before Stronghold Agent becomes a production implementation phase, freeze and validate at minimum:

```text
supported Windows versions / editions
service lifecycle
installer / uninstall behavior
binary / installer signing
update / rollback / anti-downgrade
endpoint enrollment / identity
certificate / key protection
Stronghold Access control protocol
policy bundle schema / signing
policy generation / lease / holdover
network-context fact source/generation
WFP integration / layer selection
filter ownership / cleanup
application / process identity semantics
user / device identity semantics
posture scope / freshness
network-context authority
Network Access Session correlation
Protected Endpoint transport contract
full-tunnel bypass/leak matrix
WireGuard integration
Agent / FW dual-PEP semantics
endpoint decision record schema
record integrity / transport
local-admin tamper boundary
performance / latency / resource use
sleep / resume / network transition behavior
multi-NIC behavior
other-VPN coexistence policy
failure / recovery behavior
Hunter correlation
health / observability
```

Before any deep payload-aware endpoint inspection is added, separately freeze and validate:

```text
why connection/process identity is insufficient
selected proxy or WFP callout-driver architecture
protocol coverage
TLS handling
kernel/user boundary
code/driver signing
Windows-version compatibility
crash / BSOD containment
performance / latency
privacy / data handling
logging / retention
support diagnostics
failure behavior
```

## Engineering Principle

> **Stronghold Agent knows the endpoint side of the connection. Stronghold FW knows the network side. Stronghold Access coordinates authorization without pretending those are the same source of truth.**
