# Stronghold Access Architecture

## Purpose

**Stronghold Access** is the Stronghold platform's access-control and authorization-coordination infrastructure component.

Stronghold Access is **not a separate standalone commercial product**. It is the third Stronghold infrastructure node when deployed, running on a supported customer virtual machine or supported bare-metal server on the customer's network.

Stronghold Access is deliberately kept out of the high-PPS Stronghold FW dataplane, but it is tightly coupled to the Stronghold platform through explicit authenticated/versioned contracts.

Stronghold Access owns access-control responsibilities such as:

```text
identity inputs
device trust
endpoint posture
network admission / 802.1X
AAA integration
Stronghold Access Sessions
resource-scoped authorization
revocation / reevaluation
policy distribution to Stronghold Agent
controlled integration with Stronghold FW
optional Pathfinder intelligence / risk inputs
```

Stronghold Access does **not** own Stronghold FW packet capture, routing, NAT, dataplane forwarding, physical-interface observation, FW Traffic Decision Journals, or Net-Hunter authoritative packet history.

## Platform Boundary

```text
                         STRONGHOLD PLATFORM

Stronghold FW
    physical observation / enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    third infrastructure component
    customer VM or bare-metal server
    control / identity / authorization coordination

Stronghold Agent
    endpoint component / endpoint PEP
```

Stronghold Access and Stronghold Agent remain separate implementation trees inside the Stronghold repository, but those implementation boundaries do not make them unrelated products.

They communicate through explicitly defined, versioned Stronghold contracts rather than direct imports of one another's internal implementation packages.

See `docs/PROJECT-BOUNDARIES.md`.

## Deployment Model

Stronghold Access is intended to run on the customer's network as either:

```text
supported customer virtual machine
or
supported customer bare-metal server
```

Conceptually:

```text
                    STRONGHOLD ACCESS
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
   Stronghold Agent    RADIUS / AAA     Stronghold FW
    endpoint PEP       802.1X context    network PEP
          |                |                |
          +----------------+----------------+
                           |
                           v
                   authorization context
```

Stronghold Access is control-plane infrastructure, not the Stronghold FW high-rate packet dataplane. Its sizing and qualification should therefore be based on endpoint population, authentication/authorization rate, AAA rate, policy distribution, revocation rate, session count, state/database behavior, and availability requirements rather than packet PPS.

Exact supported host OS, hypervisors, database/storage model, HA model, backup/recovery model, and sizing profiles remain to be frozen.

## Stronghold FW Integration

Stronghold Access integrates with Stronghold FW through an authenticated, narrowly scoped Stronghold control contract.

```text
Stronghold Access
       |
       | mTLS / approved authenticated transport
       | explicit peer authorization
       | versioned control contract
       v
Stronghold FW
       |
       v
network-side PEP
```

Access does not receive unrestricted shell, root, package-management, filesystem, or arbitrary command authority over Stronghold FW.

Potential control facts include:

```text
Access Session create / update / revoke
subject identity
device identity
device posture generation
application / process identity where established
resource / destination authorization
service / protocol authorization
session expiration
policy generation
revocation reason
network-context generation
Pathfinder-derived risk context where policy permits
```

The exact wire schema, signing, replay protection, sequencing, lease behavior, and failure handling must be frozen before implementation.

## Governing Authorization Boundaries

> **Network admission establishes permission to attach. It does not establish permission to access resources.**

> **Path availability is not path permission.**

> **External AAA may establish identity or return authorization attributes; it never bypasses Stronghold policy.**

> **An Access GRANT does not force Stronghold FW to forward traffic.**

> **Endpoint ALLOW never grants permission that Stronghold FW would otherwise deny.**

> **Pathfinder intelligence is an input to policy, not policy authority by itself.**

Mandatory separations include:

```text
802.1X authenticated                 != resource authorized
RADIUS Access-Accept                 != Stronghold Access GRANT
AAA user authenticated               != device trusted
device trusted                       != posture acceptable
posture acceptable                   != process authorized
process authorized                   != resource authorized
Access GRANT                         != FW ALLOW
endpoint ALLOW                       != FW ALLOW
endpoint DENY                        != FW observed DENY
WireGuard peer authenticated         != user authenticated
tunnel established                  != resource authorized
network assignment                   != trusted endpoint
MAB admitted                         != 802.1X authenticated
Pathfinder risk signal               != automatic Access REVOKE
Pathfinder malicious classification  != compromise proven
```

## First-Class 802.1X Network Admission

802.1X is a first-class Stronghold Access security primitive, not an optional RADIUS checkbox added after the policy model is complete.

Stronghold Access should model network admission explicitly.

First-class concepts include:

```text
AAA Provider
Authenticator
Network Access Policy
Network Access Session
802.1X Identity
EAP Method
Certificate Identity
Authentication Result
Authorization Result
Network Assignment
CoA / Reauthentication
Session Termination
Admission Health
```

Conceptually:

```text
ENDPOINT
   |
   | EAP / 802.1X
   v
SWITCH / AP
Authenticator
   |
   | RADIUS
   v
NPS / ISE / ClearPass / FreeRADIUS / qualified AAA
   |
   v
NETWORK ACCESS SESSION
   |
   +-- endpoint / subject identity
   +-- authentication method
   +-- certificate identity where applicable
   +-- authenticator identity
   +-- switch / AP
   +-- port / SSID where available
   +-- VLAN / role / network assignment
   +-- session state
   +-- authorization result
   |
   v
STRONGHOLD ACCESS
```

Stronghold should consume and preserve the strongest facts the network-admission system can actually establish without manufacturing certainty that the source did not provide.

### Preferred Authenticated Admission

EAP-TLS is an important high-assurance direction where supported by the customer's identity and certificate infrastructure.

Exact supported EAP methods, certificate requirements, revocation behavior, TLS profiles, RADIUS attributes, dynamic authorization behavior, and vendor interoperability are qualification work.

### MAB Boundary

MAC Authentication Bypass may be necessary for printers, phones, cameras, embedded devices, and legacy systems, but it is not equivalent to authenticated 802.1X.

```text
MAB admitted
!=
802.1X authenticated
```

Stronghold policy must be able to distinguish the assurance level and apply appropriately restricted network/resource policy.

## AAA Integration

Stronghold Access should integrate with existing enterprise AAA rather than require replacement of established infrastructure.

Potential supported systems include, subject to qualification:

```text
Microsoft NPS
RADIUS servers
Cisco ISE
Aruba ClearPass
FreeRADIUS
other standards-compliant RADIUS platforms
```

TACACS+ belongs primarily to administrative/device AAA use cases and must not be conflated with 802.1X network admission.

Stronghold may also consume identity/group facts through approved directory or identity-provider integrations.

Returned AAA attributes are inputs to Stronghold authorization. They do not become hidden policy that bypasses the Stronghold Policy Engine.

## Three Session Types

Stronghold Access preserves three different session concepts.

### Network Access Session

Answers:

> Why is this endpoint permitted to attach to this network?

Potential facts:

```text
802.1X / MAB method
authenticator
RADIUS / AAA source
EAP method
certificate identity
subject / machine identity
network assignment
session start / end
CoA / reauthentication state
```

### Stronghold Access Session

Answers:

> What may this subject/device/application access right now?

Potential facts:

```text
Access Session ID
subject / user identity
device identity
device certificate / trust state
MFA state
posture state / posture generation
application / process identity where established
Policy Generation
requested resource
protocol / service
grant scope
created time
expiration time
authorization state
revocation reason
```

### Transport Session

Answers:

> How is protected traffic being carried?

Potential facts:

```text
WireGuard peer identity
tunnel/session identity
tunnel address
cryptographic/session lifetime
transport health
```

Mandatory separation:

```text
Network Access Session
!=
Stronghold Access Session
!=
Transport Session
```

## Stronghold Agent Relationship

Stronghold Agent is the endpoint-side Stronghold component and endpoint Policy Enforcement Point.

Stronghold Access remains the control framework / policy authority.

```text
Stronghold Access
       |
       | signed / authenticated policy and session state
       v
Stronghold Agent
       |
       v
endpoint PEP
```

The Agent may establish endpoint-origin facts such as user/device/application/process context and enforce locally before traffic reaches the network.

The Agent is not an independent Stronghold Policy Engine and must not silently grant access that Access or Stronghold FW would otherwise deny.

See `docs/agent/ARCHITECTURE.md`.

## Network Context and Endpoint Policy Distribution

Stronghold FW and qualified network-admission sources can contribute network-side facts that the endpoint itself cannot authoritatively self-assert, including observed VLAN, zone, interface, address association, and other qualified attachment context.

The intended control flow is:

```text
Stronghold FW / network admission
    establishes network-side context
        |
        | authenticated/versioned fact exchange
        v
Stronghold Access
    correlates endpoint + network context
    evaluates applicable policy
        |
        | signed monotonic endpoint policy generation
        v
Stronghold Agent
    verifies / validates / programs WFP
        |
        v
local endpoint PEP
```

The endpoint does not become trusted merely because it reports that it is on a particular VLAN or network.

```text
Agent-reported VLAN / zone
!=
FW- or admission-established VLAN / zone

network context received
!=
endpoint policy activated

endpoint policy activated
!=
endpoint enforcement healthy
```

Policy distribution should be generation-based and bounded rather than a chatty per-packet control protocol. A network-context change may cause Access to reevaluate and publish a new endpoint policy generation, but live packet forwarding on Stronghold FW must not depend on synchronous endpoint-policy delivery.

## Local Endpoint Enforcement Assistance

Stronghold Access may authorize Stronghold Agent to reject disallowed application/process connections locally before those connections reach the LAN, WireGuard transport, or Stronghold FW.

This provides an additional enforcement point close to the originating process and can reduce unnecessary work presented to the network firewall.

Conceptually:

```text
process connection request
        |
        v
Stronghold Agent / WFP PEP
        |
   +----+----+
   |         |
 DENY      ALLOW
   |         |
local        v
record    network path
             |
             v
        Stronghold FW
             |
             v
       independent policy
```

The optimization never changes authority:

```text
endpoint rejected traffic
!=
FW denied traffic

endpoint permitted traffic
!=
FW authorized traffic
```

The first Windows direction is process/application-aware connection authorization, not a requirement for endpoint deep-payload inspection. Any later payload-aware proxy or WFP callout-driver architecture is separately qualified.

## Pathfinder Intelligence / Risk Input

Stronghold Access may consume approved Pathfinder intelligence as one input to authorization and reevaluation.

Pathfinder remains a separate ISS threat-intelligence authority. It does not become the Stronghold Policy Engine.

Potential Pathfinder-derived inputs may include:

```text
observable classification
confidence
freshness / age
campaign or malware association
Pathfinder Record ID
interpretation generation/version
source provenance
relationship to a Stronghold endpoint/session/resource
```

An example flow is:

```text
Stronghold observation / IDS context
        |
        v
Pathfinder enrichment
        |
        v
Stronghold Access reevaluation
        |
        +-- maintain authorization
        +-- require MFA reauthentication
        +-- reduce resource scope
        +-- shorten lease
        +-- REVOKE
        +-- request network-admission reevaluation / CoA
```

The result is controlled by Stronghold Access policy.

```text
Pathfinder risk signal
!=
automatic Access Session revocation

Pathfinder match
!=
endpoint compromise proven
```

When Pathfinder materially influences an Access decision, the Access record/journal lineage should retain the Pathfinder Record ID and interpretation/version used.

See `docs/PATHFINDER-INTEGRATION.md`.

## Protected Endpoint Integration

Stronghold Access may place selected endpoints into a **Protected Endpoint** mode implemented by Stronghold Agent.

This may be appropriate for high-value systems such as privileged administration workstations, finance systems, CJIS workstations, domain infrastructure, backup infrastructure, critical servers, or other explicitly selected endpoints.

In Protected Endpoint mode, eligible endpoint traffic is forced through an authenticated encrypted Stronghold transport toward the authorized Stronghold enforcement point.

The local network may still observe link-layer and encrypted-flow metadata. Protected Endpoint does not claim to make the endpoint physically invisible.

```text
cannot decrypt protected payload
!=
cannot observe traffic metadata

cannot read protected traffic
!=
cannot disrupt transport
```

Exact bypass/failure behavior is defined by the Agent architecture and later qualification.

## Multiple Enforcement Points

A mature deployment may include three distinct enforcement locations:

```text
NETWORK-ADMISSION PEP
    switch / AP / 802.1X

ENDPOINT PEP
    Stronghold Agent

NETWORK PEP
    Stronghold FW
```

A decision at one location does not fabricate a decision at another.

Example:

```text
endpoint denied before transmission
!=
FW denied packet
```

If the FW never saw the traffic, Stronghold must not create a false FW observation.

## Revocation and Dynamic Authorization

Revocation is a first-class lifecycle operation.

Potential triggers include:

```text
account disabled
device certificate revoked
MFA/session expiration
posture failure
policy generation change
administrative revoke
device compromise state
Access Session expiration
network-admission state change
network-context change
approved Pathfinder intelligence/risk change
```

Where supported and authorized, RADIUS Dynamic Authorization / Change of Authorization may be used to request reauthentication, quarantine, or disconnect at the network-admission layer.

Conceptually:

```text
posture / trust / network context / intelligence changes
        |
        v
Stronghold Access reevaluates
        |
        v
REVOKE / NEW POLICY GENERATION
   +----+-------------------+
   |                        |
   v                        v
Agent PEP                 FW PEP
   |
   +---- optional CoA ----> Network Admission PEP
```

Stronghold must distinguish requested CoA from confirmed network-side effect.

## Stronghold FW Relationship

Stronghold FW remains an independent network policy and enforcement authority.

```text
Access Session valid
!=
FW policy allows

FW policy allows
!=
forwarding succeeded
```

The FW may consume authenticated Access Session state as a policy fact. It still applies normal Stronghold policy, routing, NAT, path, tunnel, and enforcement semantics.

Stronghold Access must never become a hidden route around normal FW policy.

## Net-Hunter Relationship

Net-Hunter may later correlate:

```text
Network Access Session
Stronghold Access Session
Stronghold Agent endpoint decision
Stronghold FW decision
WireGuard transport/session state
network-context generation
Pathfinder Record / interpretation reference
related authoritative packet segments
```

Correlation is derived interpretation and does not rewrite source authority.

Net-Hunter is not required for Access to make routine authorization decisions.

## Availability and Failure Boundaries

Stronghold Access is security-sensitive control-plane infrastructure. Its failure model must be explicit.

Potential states include:

```text
AVAILABLE
DEGRADED
HOLDOVER
AAA_UNAVAILABLE
POLICY_UNAVAILABLE
PATHFINDER_UNAVAILABLE
PATHFINDER_STALE
REVOCATION_DEGRADED
CONTROL_CHANNEL_UNAVAILABLE
RECOVERY
```

Exact fail-open/fail-closed behavior must be scoped by use case rather than reduced to one global behavior.

A failure must not silently convert:

```text
unknown
```

into:

```text
trusted
```

Pathfinder unavailability must not be represented as a safe intelligence result.

## Implementation Boundary

Stronghold Access implementation belongs under:

```text
go/access/
```

Stronghold Agent implementation belongs under:

```text
go/agent/
```

The two components must not directly import one another's internal implementation packages.

Do not create a generic shared framework to make the separation disappear. Shared protocol/schema code may be introduced only after a real versioned contract requires it and its ownership is explicit.

## Qualification Requirements

Before Stronghold Access becomes a production implementation phase, freeze and validate at minimum:

```text
supported host OS / deployment profiles
supported VM platforms / bare-metal profile
identity provider model
AAA provider model
802.1X / EAP interoperability matrix
Network Access Session schema
RADIUS attribute handling
CoA / reauthentication semantics
MAB assurance boundary
Stronghold Access Session schema
application/process binding semantics
MFA model
posture model
policy language / generations
network-context fact contract from FW/admission
Agent control protocol
FW control protocol
Pathfinder intelligence/risk input contract
Pathfinder freshness/holdover semantics
mTLS / trust / certificate lifecycle
replay / downgrade protection
session lease / expiration behavior
revocation semantics
FW-integrated failure behavior
HA / recovery / backup
journaling / audit records
observability / external monitoring
scale / latency / load qualification
upgrade / rollback
```

## Engineering Principle

> **Stronghold Access coordinates authorization across the Stronghold platform. It does not erase the independent authority of endpoint enforcement, network admission, Stronghold FW, or Pathfinder's threat-intelligence interpretation.**
