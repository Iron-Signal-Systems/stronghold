# Stronghold Access Architecture

## Purpose

**Stronghold Access** is the Stronghold access-control product and control framework.

Stronghold Access is deployable independently of Stronghold FW and may run on a supported customer virtual machine or supported bare-metal host. When Stronghold FW is present, Access integrates with it as a first-class Stronghold control-plane peer rather than becoming part of the FW process or operating-system image.

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
```

Stronghold Access does **not** own Stronghold FW packet capture, routing, NAT, dataplane forwarding, physical-interface observation, FW Traffic Decision Journals, or Net-Hunter authoritative packet history.

## Product Boundary

```text
STRONGHOLD PLATFORM

Stronghold FW
    physical enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    VM or bare-metal access-control system

Stronghold Agent
    endpoint application / endpoint PEP
```

Stronghold Access and Stronghold Agent are separate implementation projects inside the Stronghold repository.

Their implementation trees must remain separate. They communicate through explicitly defined, versioned Stronghold contracts rather than direct imports of one another's internal implementation packages.

## Deployment Models

### Standalone

Stronghold Access may be deployed without Stronghold FW.

```text
                 STRONGHOLD ACCESS
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
   Stronghold Agent   RADIUS/AAA   Switch / AP
      endpoint PEP                  802.1X PEP
```

Standalone deployment may provide endpoint identity, posture, process/application-aware endpoint enforcement, network-admission integration, Access Session control, revocation, and related policy services.

Standalone Access does not claim Stronghold FW physical-interface packet history, Stronghold FW network-side enforcement, or Stronghold FW decision-journal authority when no Stronghold FW is present.

### Stronghold FW Bolt-On

When Stronghold FW is present, Access integrates through an authenticated, narrowly scoped control interface.

```text
Stronghold Access
       |
       | mTLS
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
```

The exact wire schema, signing, replay protection, sequencing, lease behavior, and failure handling must be frozen before implementation.

## Governing Authorization Boundaries

> **Network admission establishes permission to attach. It does not establish permission to access resources.**

> **Path availability is not path permission.**

> **External AAA may establish identity or return authorization attributes; it never bypasses Stronghold policy.**

> **An Access GRANT does not force Stronghold FW to forward traffic.**

Mandatory separations include:

```text
802.1X authenticated             != resource authorized
RADIUS Access-Accept             != Stronghold Access GRANT
AAA user authenticated           != device trusted
device trusted                   != posture acceptable
posture acceptable               != process authorized
process authorized               != resource authorized
Access GRANT                     != FW ALLOW
WireGuard peer authenticated     != user authenticated
tunnel established              != resource authorized
network assignment               != trusted endpoint
MAB admitted                     != 802.1X authenticated
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

Stronghold Access should preserve three different session concepts.

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

Stronghold Agent is the endpoint application and endpoint Policy Enforcement Point.

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
```

Where supported and authorized, RADIUS Dynamic Authorization / Change of Authorization may be used to request reauthentication, quarantine, or disconnect at the network-admission layer.

Conceptually:

```text
posture / trust changes
        |
        v
Stronghold Access reevaluates
        |
        v
REVOKE
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

When Access is integrated, the FW may consume authenticated Access Session state as a policy fact. It still applies normal Stronghold policy, routing, NAT, path, tunnel, and enforcement semantics.

Stronghold Access must never become a hidden route around normal FW policy.

## Net-Hunter Relationship

Net-Hunter may later correlate:

```text
Network Access Session
Stronghold Access Session
Stronghold Agent endpoint decision
Stronghold FW decision
WireGuard transport/session state
related authoritative packet segments
```

Correlation is derived interpretation and does not rewrite source authority.

Net-Hunter is not required for Access to make routine authorization decisions unless a later explicit design changes that boundary.

## Availability and Failure Boundaries

Stronghold Access is security-sensitive control-plane infrastructure. Its failure model must be explicit.

Potential states include:

```text
AVAILABLE
DEGRADED
HOLDOVER
AAA_UNAVAILABLE
POLICY_UNAVAILABLE
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

## VM and Bare-Metal Deployment

Stronghold Access is intended to support qualified deployment as either:

```text
customer virtual machine
or
customer bare-metal host
```

Stronghold Access is control-plane infrastructure, not the Stronghold FW high-rate packet dataplane. Its hardware/performance qualification should therefore be based on control-plane workloads such as endpoint population, authorization rate, AAA rate, policy distribution, revocation rate, session count, database/storage behavior, and availability requirements.

Exact supported hypervisors, host operating system, database/storage model, HA model, sizing profiles, backup/recovery model, and appliance/package lifecycle remain to be frozen.

## Implementation Boundary

Stronghold Access implementation belongs under:

```text
go/access/
```

Stronghold Agent implementation belongs under:

```text
go/agent/
```

The two projects must not directly import one another's internal implementation packages.

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
Agent control protocol
FW bolt-on control protocol
mTLS / trust / certificate lifecycle
replay / downgrade protection
session lease / expiration behavior
revocation semantics
standalone failure behavior
FW-integrated failure behavior
HA / recovery / backup
journaling / audit records
observability / external monitoring
scale / latency / load qualification
upgrade / rollback
```

## Engineering Principle

> **Stronghold Access establishes and evaluates authorization context. It does not erase the independent authority of the endpoint, the network-admission system, or Stronghold FW.**
