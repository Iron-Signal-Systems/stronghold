# Stronghold Distributed Policy Enforcement

## Purpose

This document defines the cross-component Stronghold contract for synchronized policy authority, device-specific endpoint policy projection, early endpoint enforcement, off-network continuity, policy-generation synchronization, and truthful cross-PEP reporting.

Stronghold is one enforcement platform with multiple Policy Enforcement Points (PEPs). The existence of multiple PEPs does not create multiple unrelated policy authorities.

> **A finalized Stronghold policy generation is the common policy authority. Each enforcement point receives and enforces the portion of that generation applicable to its responsibility.**

> **Stronghold Agent may deny an unauthorized endpoint connection before a packet reaches the network, but it must never fabricate a Stronghold FW observation or FW denial for traffic that was never presented to the FW.**

> **A managed endpoint remains subject to its active Stronghold endpoint policy when it leaves the organization's network.**

This contract is intentionally separate from implementation details such as the eventual policy language, wire schema, database, WFP layer selection, or management UI.

## Component Roles

```text
STRONGHOLD POLICY / CONFIGURATION AUTHORITY
    finalized policy generation
            |
            +------------------------------+
            |                              |
            v                              v
    STRONGHOLD ACCESS                STRONGHOLD FW
    control / coordination           network PEP
            |
            | device-specific
            | endpoint projection
            v
    STRONGHOLD AGENT
    endpoint PEP
            |
            +---- DENY locally
            |
            +---- ALLOW attempt
                     |
                     v
                network path
                     |
                     v
                STRONGHOLD FW
                independent network enforcement

NET-HUNTER
    historical correlation / preservation of applicable source records
```

Responsibilities remain distinct:

```text
Stronghold policy/configuration authority
    finalize governed policy generations

Stronghold Access
    coordinate identity, device, posture, network context, Access Sessions,
    policy projection/distribution, synchronization state, revocation,
    Agent check-in, and cross-component authorization context

Stronghold Agent
    establish endpoint-origin facts and enforce the endpoint-applicable
    policy projection before unauthorized traffic leaves the endpoint

Stronghold FW
    observe traffic actually presented to supported physical interfaces
    and independently enforce network policy

Net-Hunter
    preserve/correlate historical records without rewriting source authority
```

## One Finalized Generation, Multiple Enforcement Projections

Stronghold must not create an unrelated endpoint policy model that drifts independently from network policy.

A finalized Stronghold generation may contain policy relevant to several enforcement locations. Stronghold Access derives the endpoint-applicable projection for each managed device from that same finalized authority.

Conceptually:

```text
FINALIZED GENERATION 844
        |
        +---- Stronghold FW projection
        |
        +---- FIN-PC-17 Agent projection
        |
        +---- HR-PC-22 Agent projection
        |
        +---- CJIS-PC-04 Agent projection
```

The projections do not need to be byte-for-byte identical because the enforcement points have different responsibilities.

For example, Stronghold FW may require policy/state for:

```text
interfaces
zones
VLANs
routing
NAT
WAN selection
HA
site tunnels
unmanaged devices
cross-zone traffic
```

while an endpoint projection may require:

```text
user / subject scope
device scope
application / process scope
resource / destination scope
service / protocol
authorization action
policy identity
policy generation
validity / lease state
network-context requirements
Protected Endpoint requirements
```

Mandatory separation:

```text
same policy authority
!=
identical enforcement representation

Agent policy projection
!=
complete FW configuration

Agent ALLOW
!=
FW ALLOW
```

An Agent ALLOW means the endpoint policy permits the connection attempt to leave the endpoint. Stronghold FW remains independently authoritative for traffic that is actually presented to it.

## Early Endpoint DENY

Where the active endpoint policy definitively denies a connection, Stronghold Agent should reject it locally before the packet is transmitted.

Example:

```text
Policy:       798
Generation:   844
Device:       FIN-PC-17
User:         DOMAIN\John
Application:  powershell.exe
Destination:  PAYROLL-DB
Service:      TCP/1433
Action:       DENY
```

Runtime behavior:

```text
powershell.exe
      |
      | connect PAYROLL-DB:1433
      v
Stronghold Agent
      |
      | Generation 844 / Policy 798
      v
    DENY
      |
      +---- endpoint decision record
      |
      +---- no packet transmitted
```

This is not merely duplicate filtering. It prevents known unauthorized managed-endpoint traffic from consuming unnecessary network and firewall work.

Potential work avoided at Stronghold FW includes work that would otherwise be associated with receiving and processing the denied traffic, such as session/state handling, policy evaluation, routing/NAT consideration, and other dataplane processing.

Stronghold must not overstate the optimization. Only traffic actually prevented from transmission by the Agent is absent from the network path.

## Truthful Endpoint-DENY Reporting

A local endpoint denial must remain attributable to the endpoint enforcement point.

Conceptual record:

```text
Endpoint Decision ID:   ED-...
Device:                 FIN-PC-17
User:                   DOMAIN\John
Application:            powershell.exe
Destination:            PAYROLL-DB
Service:                TCP/1433
Policy Generation:      844
Policy:                 798
Decision:               DENY
Enforcement Point:      STRONGHOLD_AGENT
Packet Transmitted:     NO
```

The Stronghold operator view may surface this event alongside FW activity, but it must preserve the source truth.

```text
DENIED AT ENDPOINT
Policy 798
Packet presented to FW: NO
```

It must not be rewritten as:

```text
FW DENIED
```

when the FW never received the packet.

Mandatory separations:

```text
endpoint DENY
!=
FW DENY

endpoint decision reported to platform
!=
packet presented to FW

FW has knowledge of endpoint attempt
!=
FW observed endpoint packet
```

The exact transport for endpoint decision records is a control/record contract, not fabricated dataplane traffic. Stronghold Access is the natural coordination point for Agent-origin records and operator correlation.

## Policy Finalization and Distribution

A policy edit does not become endpoint authority merely because it exists as a candidate.

Conceptually:

```text
RUNNING GENERATION 843
        |
EDIT CANDIDATE
        |
VALIDATE / DIFF / SIMULATE
        |
FINALIZE / AUTHORIZE
        |
GENERATION 844
        |
        +---- activate applicable FW state
        |
        +---- publish applicable Agent projections
```

Only the finalized/authorized generation is eligible for normal activation and distribution.

Stronghold Access should track, at minimum conceptually:

```text
device identity
required/current policy generation
last acknowledged generation
last check-in
policy delivery state
policy verification state where reported
policy activation state where reported
policy lease / expiration state
Agent health / enforcement state
```

The exact schema remains implementation work.

## Push Plus Check-In Catch-Up

Stronghold should attempt timely distribution when a new generation is finalized, but successful push delivery must never be assumed merely because distribution was attempted.

Conceptually:

```text
Generation 844 finalized
        |
        +---- push/update available to connected Agents
        |
        +---- record devices not current
```

If an Agent is offline, asleep, disconnected, or otherwise misses the push, its next authenticated check-in must compare its active generation with the currently required generation.

Example:

```text
Stronghold required generation:  844
FIN-PC-17 active generation:      843

FIN-PC-17 check-in
        |
        v
Stronghold Access detects mismatch
        |
        v
send signed device projection for 844
        |
        v
Agent verify / validate / stage / activate
        |
        v
Agent reports resulting state
```

Mandatory separations:

```text
policy generation finalized
!=
every Agent synchronized

policy sent
!=
policy received

policy received
!=
policy verified

policy verified
!=
policy activated

policy activated
!=
enforcement healthy
```

Stronghold must make generation drift visible rather than silently assuming fleet convergence.

## Off-Network Enforcement Continuity

A central purpose of the synchronized Agent policy is to preserve applicable Stronghold restrictions when a managed device is no longer behind Stronghold FW.

Conceptually:

```text
FIN-PC-17
    office LAN      -> Agent Generation 844
    home network    -> Agent Generation 844
    hotel network   -> Agent Generation 844
    mobile hotspot  -> Agent Generation 844
```

If Policy 798 prohibits an application/resource connection for FIN-PC-17, leaving the organization's network does not make Policy 798 disappear.

The active endpoint projection continues to govern local connection attempts according to its explicit validity/lease/holdover contract.

```text
on-network endpoint restriction
!=
restriction only when FW is physically in path
```

This does not mean Stronghold Agent replaces Stronghold FW. Off-network, the Agent enforces what the endpoint can establish and control locally. Network-side facts and network-side enforcement that require Stronghold FW remain unavailable unless the endpoint uses an authorized Stronghold protected/remote-access path.

## Stale, Holdover, and Expired Policy

Stronghold must not define synchronization failure as either unlimited trust forever or automatic destruction of all connectivity.

Potential endpoint policy states include:

```text
POLICY_CURRENT
POLICY_HOLDOVER
POLICY_STALE
POLICY_EXPIRED
POLICY_REVOKED
CONTROL_UNAVAILABLE
```

Exact names and transitions remain to be frozen.

Policy profiles may have different holdover/failure semantics. A normal managed workstation may be allowed a bounded period under the last verified generation, while a Protected Endpoint or privileged system may intentionally use stricter expiration behavior.

The governing requirements are:

```text
unknown
!=
trusted

last known policy
!=
valid forever

control unavailable
!=
automatically allow everything

control unavailable
!=
automatically disable all networking
```

## Application / Process Identity

Stronghold Agent may know the originating Windows process before traffic is encrypted or transmitted.

That endpoint-origin fact is distinct from network-derived application interpretation.

```text
originating process identified
!=
payload decoded

Windows process identity
!=
network App-ID classification
```

Example:

```text
Endpoint fact:
    OUTLOOK.EXE initiated the connection

Network interpretation:
    traffic classified as Microsoft 365 / Exchange
```

Both may be useful and both may be true, but neither should overwrite the other.

For managed endpoints, process-aware authorization allows Stronghold to answer a question that pure network classification cannot answer with the same source authority:

> Which local process actually requested this connection, and was that process permitted to attempt it?

For unmanaged endpoints, Stronghold FW must stand on network-side facts and network-side classification because no trusted Agent process fact exists.

## Network Admission and Access Context

Stronghold Access may correlate the endpoint projection with network-admission and network-side facts such as:

```text
802.1X / MAB state
authenticator
user / machine identity
device certificate
VLAN / zone
attachment context
posture generation
Stronghold Access Session
```

These facts may affect the policy projection, but none creates hidden bypass authority.

```text
802.1X authenticated
!=
resource authorized

Agent ALLOW
!=
FW ALLOW

Access GRANT
!=
FW ALLOW
```

## Protected Endpoint Relationship

Protected Endpoint mode may use the same synchronized authorization model while forcing eligible traffic through an authenticated encrypted Stronghold transport.

Conceptually:

```text
high-value endpoint
        |
Stronghold Agent
    local policy PEP
        |
        | authorized encrypted transport
        v
Stronghold FW
    independent network PEP
```

The Agent still evaluates endpoint-applicable policy before transmission. The protected transport does not turn a denied connection into an allowed one.

```text
tunnel available
!=
connection authorized
```

## Resource and Performance Intent

Early Agent denial has a deliberate resource-protection benefit.

Stronghold FW should devote its high-rate dataplane resources primarily to traffic that actually reaches the network enforcement point, including unmanaged endpoints, inbound traffic, cross-zone traffic, routing/NAT/WAN functions, observation, state, and inspection.

Managed endpoint traffic that can be definitively rejected at the endpoint does not need to be intentionally sent to the FW merely so the FW can reject it again.

This optimization never permits the Agent to broaden authorization. The endpoint PEP can reduce work by rejecting earlier; it cannot force the network PEP to accept later.

## Net-Hunter Correlation

Net-Hunter may correlate, where available:

```text
Endpoint Decision ID
Device ID
user / subject
application / process
Agent policy generation
policy identity
Stronghold Access Session
network-context generation
FW decision
FW traffic/segment references
Protected Endpoint transport
Pathfinder interpretation references
```

Correlation must preserve source authority.

For an endpoint-local DENY:

```text
endpoint decision exists
FW packet observation does not exist
```

That is a valid complete history. Stronghold must not treat the absence of a FW packet as a missing record when the Agent truthfully prevented transmission.

## Governing Invariants

```text
one finalized Stronghold policy authority
multiple enforcement projections

Agent policy projection
!=
independent endpoint policy universe

Agent DENY
=
traffic must not be transmitted by that authorized endpoint path

Agent ALLOW
!=
FW ALLOW

endpoint DENY reported to Stronghold
!=
FW observed packet

policy push attempted
!=
fleet synchronized

Agent off corporate network
!=
Agent policy disabled

same finalized generation
!=
identical representation at every PEP
```

## Engineering Principle

> **Stronghold moves enforcement as close to the originating action as practical without moving or fabricating authority. The Agent can stop known-denied traffic before it becomes network traffic; Access keeps the endpoint synchronized with the finalized Stronghold policy; FW independently controls what is actually presented to the network; and Net-Hunter preserves the history without pretending one enforcement point acted for another.**
