# Stronghold Secure Access Architecture

## Purpose

This document defines the current future architecture for Stronghold secure remote access and site-to-site / branch-office tunneling.

**Secure-access implementation remains deferred.** This document establishes the architectural boundaries that any later implementation must preserve. It does not pull VPN or secure-access work into Phase 0.

Stronghold deliberately separates two different responsibilities:

```text
STRONGHOLD ZERO-TRUST REMOTE ACCESS
    user/device/resource authorization
    NIST SP 800-207 control-model direction
    WireGuard secure transport/data plane

STRONGHOLD SITE-TO-SITE / BRANCH OFFICE
    site/network/tunnel relationship
    L2TPv3 tunnel
    IPsec protection
    Stronghold bridge/routing/policy integration
```

These are different products/contracts. They may share Stronghold identity, management, observability, journaling, and enforcement primitives, but they are not represented as one generic VPN session model.

The governing Stronghold principle remains:

> **Path availability is not path permission.**

## Zero-Trust Remote Access

The remote-access architecture follows the NIST SP 800-207 separation between control-plane authorization and data-plane enforcement.

Conceptually:

```text
                         STRONGHOLD ZERO-TRUST ACCESS

                         CONTROL PLANE

 Identity / MFA / Device Identity / Posture / Policy / PKI
             │
             ▼
     ┌─────────────────┐
     │  POLICY ENGINE  │
     │       PE        │
     │                 │
     │ GRANT           │
     │ DENY            │
     │ REVOKE          │
     └────────┬────────┘
              │ decision
              ▼
     ┌─────────────────┐
     │ POLICY ADMIN    │
     │       PA        │
     │                 │
     │ Creates/removes │
     │ access session  │
     └────────┬────────┘
              │
              │ programs
              ▼
═══════════════════════════════════════════════════════════════════════

                          DATA PLANE

 Remote Endpoint
 ┌────────────────┐
 │ Stronghold     │
 │ Access Agent   │
 │                │
 │ WireGuard      │
 └───────┬────────┘
         │
         │ encrypted WireGuard transport
         ▼
 ┌───────────────────────────┐
 │      STRONGHOLD FW        │
 │                           │
 │ Policy Enforcement Point  │
 │           PEP             │
 │                           │
 │ WireGuard termination     │
 │ identity/session binding  │
 │ resource authorization    │
 │ normal FW enforcement     │
 │ traffic observation       │
 │ decision journaling       │
 └─────────────┬─────────────┘
               │
               │ explicitly authorized
               ▼
        Enterprise Resource
```

## WireGuard Boundary

WireGuard is the core secure transport/data-plane mechanism for Stronghold zero-trust remote access.

> **WireGuard proves and protects the encrypted peer-to-peer transport. Stronghold decides what that authenticated transport may access.**

WireGuard peer identity and `AllowedIPs`/cryptokey-routing behavior are useful transport primitives, but they are not the Stronghold authorization system.

Mandatory separations:

```text
WireGuard peer authenticated
!=
user authenticated

user authenticated
!=
device authorized

device authorized
!=
resource authorized

tunnel established
!=
resource authorized

resource authorized
!=
connection succeeded
```

Stronghold must not convert possession of a valid WireGuard key into broad network authorization.

## Stronghold Access Session

The conceptual authorization object is a Stronghold **Access Session**, not a permanently configured WireGuard peer entry.

An Access Session may bind information such as:

```text
Access Session ID
subject / user identity
device identity
device certificate / trust identity
WireGuard peer identity
tunnel address
authentication method
MFA state
posture state / posture generation
Policy Generation
requested resource
protocol / service
direction
grant scope
created time
expiration time
authorization state
revocation reason
```

Exact schema and identifiers remain to be frozen later.

A WireGuard peer/session becomes usable because the Stronghold control plane granted an Access Session. The existence of a peer is not itself the authorization decision.

## Policy Engine

The Policy Engine evaluates whether the requested subject/device/session may reach a requested resource.

Inputs may eventually include:

```text
user / subject identity
device identity
device trust/certificate state
MFA state
device posture
requested resource
service / protocol
policy generation
time / session conditions
administrative state
risk or compromise state where later supported
```

The Policy Engine produces an explicit decision such as:

```text
GRANT
DENY
REVOKE
```

The exact decision vocabulary and policy language remain implementation work.

## Policy Administrator

The Policy Administrator executes the authorized lifecycle of the access path.

Conceptually, on GRANT it may:

```text
create Access Session
bind WireGuard peer/session identity
assign scoped tunnel address
install/program permitted resource authorization
establish expiration/renewal state
journal the operation
```

On expiration or REVOKE it may:

```text
remove resource authorization
terminate associated active sessions where policy requires
remove/retire/revoke WireGuard session state as appropriate
journal the operation and result
```

The PA does not silently grant broader access than the PE authorized.

## Policy Enforcement Point

Stronghold FW acts as the data-plane Policy Enforcement Point for remote access.

The PEP is responsible for applying the active authorization to actual traffic while preserving normal Stronghold enforcement semantics.

The PEP may enforce facts such as:

```text
Access Session ID
source user/device/session binding
permitted resource / destination
service / protocol
allowed direction
session expiration
Policy Generation
normal Stronghold security policy
route / NAT / WAN behavior where applicable
```

A valid secure tunnel does not bypass normal Stronghold firewall policy.

## Stronghold Access Agent

The future Stronghold Access Agent is the endpoint-side secure-access component.

Its responsibilities may include:

```text
device identity presentation
user authentication initiation
MFA interaction
posture collection where later approved
access request creation
WireGuard secure transport
session state / expiry presentation
revocation handling
health/diagnostic information
```

The agent does not become an independent policy authority. The Stronghold control plane remains authoritative for access decisions.

Exact endpoint platforms, update mechanism, enrollment, posture collectors, and credential storage remain to be frozen.

## Authentication, Device Identity, and MFA

Remote-access authorization separates multiple identities and checks rather than collapsing them into one successful login.

Potential future inputs include:

```text
enterprise user identity
local recovery/admin identity where explicitly appropriate
device certificate / device identity
MFA result
device posture
WireGuard peer key
```

A successful user authentication does not automatically authorize an unmanaged or revoked device.

A trusted device does not automatically authorize every user or every resource.

WireGuard peer credentials are purpose-specific transport credentials and are not interchangeable with appliance identity, journal signing, HISTORY, HA, or IDS/inspection credentials.

## Resource-Scoped Authorization

Remote access is resource-scoped rather than automatically granting a remote endpoint broad routed access to an internal network.

Conceptually:

```text
subject:   DOMAIN\John
device:    LAPTOP-42
resource:  SERVER-17
service:   TCP/3389
policy:    Server-Administrators
decision:  GRANT
```

may create a scoped access session without implying authorization to unrelated networks or services.

The exact resource object model must integrate with Stronghold hosts/networks/groups/services/zones and later application/resource identity where approved.

## Revocation

Revocation is a first-class control-plane operation.

Conditions that may later trigger reevaluation or revocation include:

```text
account disabled
device certificate revoked
MFA/session expiration
device posture failure
administrative revoke
policy generation change
device compromise state
Access Session expiration
trust-state change
```

Conceptually:

```text
condition changes
      ↓
Policy Engine
      ↓
REVOKE
      ↓
Policy Administrator
      ↓
remove authorization / terminate session state
      ↓
Policy Enforcement Point denies future traffic
```

A later successful reauthentication does not erase the historical revocation event.

## Observation and Journaling

Secure-access traffic remains subject to Stronghold's physical-interface observation and capture model where presented to configured supported capture paths.

Stronghold must preserve the difference between:

```text
encrypted WireGuard outer traffic
inner authorized traffic after tunnel termination
access-control decision history
```

Exact packet-capture representation across tunnel termination remains to be frozen before implementation so Stronghold does not double-count or misrepresent what was physically observed versus what was logically processed.

Relevant journal domains may include:

```text
Trust / Identity
    authentication
    device identity
    MFA / trust state
    WireGuard peer/session identity

Administrative
    policy/session administrative changes
    manual grant/revoke where supported

Traffic Decision
    resource authorization
    actual enforcement/final result

System / Health
    PE / PA / PEP state
    tunnel/control failures
    resource exhaustion
```

## Hunter Correlation

Net-Hunter may correlate authoritative packet history and source journals with secure-access sessions.

Useful historical pivots may include:

```text
Access Session ID
user
Device ID
WireGuard peer
remote endpoint address
resource
policy generation
grant / deny / revoke
tunnel start/end
related packet segments
```

Hunter correlation remains derived interpretation. It does not rewrite FW source journals or authoritative PCAP.

Net-Hunter remains optional for basic Stronghold FW secure-access operation unless a later explicit product requirement says otherwise.

# Site-to-Site / Branch Office VPN

## Separate Architecture

Site-to-site / branch-office VPN is distinct from zero-trust remote access.

It is primarily site/network/tunnel oriented rather than user/device/resource-session oriented.

Current protocol direction:

```text
L2TPv3 tunnel
      ↓
IPsec protection
      ↓
Stronghold site/tunnel identity
      ↓
Stronghold bridge / VLAN / routing integration
      ↓
normal Stronghold policy enforcement
```

The exact IKE/IPsec profile, algorithms, authentication methods, certificate/PSK policy, rekey behavior, and interoperability matrix remain to be frozen with the Stronghold cryptographic profile and VPN qualification work.

## L2TPv3 Role

L2TPv3 provides the Layer-2 tunnel/pseudowire mechanism for the intended branch-office/site-to-site design.

Stronghold may map a qualified L2TPv3 tunnel into explicit Stronghold networking objects such as:

```text
site
site-to-site tunnel
tunnel endpoint
bridge domain
VLAN
zone
logical interface
route where applicable
```

Exact object names and whether a deployment uses transparent Layer-2 extension, routed interfaces around the pseudowire, or another approved composition remain to be frozen during implementation design.

## IPsec Role

IPsec protects the L2TPv3 transport.

IPsec establishment proves that the configured cryptographic peer relationship is active; it does not itself grant unrestricted transit through Stronghold.

Mandatory separations:

```text
L2TPv3 tunnel established
!=
site authorized

IPsec SA established
!=
traffic authorized

remote network reachable
!=
remote network permitted

tunnel healthy
!=
application connection succeeded
```

## Site and Tunnel Identity

Site-to-site configuration should preserve stable Stronghold identity separate from current endpoint addressing and cryptographic session state.

Potential concepts include:

```text
Site ID
Tunnel ID
local Appliance / Cluster ID
remote Site ID
local endpoint
remote endpoint
L2TPv3 session/pseudowire identity
IPsec peer identity
protected VLAN / bridge-domain scope
zone association
configuration generation
operational state
```

The exact model remains to be frozen later.

## Stronghold Enforcement Around Site Tunnels

A functioning site tunnel does not become an authorization shortcut.

Normal Stronghold policy continues to determine what traffic may cross the tunnel or reach local resources.

Conceptually:

```text
SITE-A-USERS
    → SITE-B-SERVERS
    permitted service set

SITE-A-GUEST
    → SITE-B
    DENY
```

The tunnel provides a secure path. Stronghold policy provides permission.

Route availability, bridge membership, tunnel health, or IPsec establishment never independently creates security authorization.

## Multi-WAN and Site Tunnels

Future site-to-site operation may integrate with Stronghold multi-WAN/path-health architecture.

A tunnel may have explicitly authorized endpoint/WAN choices, but path availability never implies permission to establish or move a tunnel to an unapproved WAN.

The exact failover/rekey/session-continuity behavior remains later qualification work.

## HA and Site Tunnels

Active/standby Stronghold HA must eventually define site-tunnel ownership, cryptographic/session-state recovery or re-establishment, virtual endpoint identity, failover timing, and truthful interruption state.

A forwarding failover does not automatically prove uninterrupted L2TPv3/IPsec continuity.

Exact state synchronization and rekey/failover behavior remain to be frozen and measured.

## Site-Tunnel Observation and Journaling

Stronghold should preserve the distinction between:

```text
outer encrypted IPsec traffic physically observed
L2TPv3 tunnel/session state
inner traffic processed after decapsulation
tunnel authorization/configuration
actual forwarding decision
```

Administrative, System/Health, Traffic Decision, Trust/Identity, and Time/Clock journals remain applicable according to the event.

Tunnel establishment, authentication, rekey, failure, recovery, administrative changes, and policy outcomes should be attributable and time-correlated.

# Management and Operational State

Both secure-access families use the normal Stronghold management authority.

CLI/API/future UI do not directly create unmanaged WireGuard, L2TPv3, or IPsec state outside Stronghold configuration generations.

Preserve:

```text
CONFIGURATION STATE
    intended secure-access configuration

OPERATIONAL STATE
    current PE/PA/PEP/tunnel/session state

HISTORICAL STATE
    prior grants/revocations/tunnel changes/failures
```

Native WireGuard/L2TPv3/IPsec state is implementation state below Stronghold. Unmanaged native changes become drift/mismatch rather than silently becoming Stronghold configuration truth.

## Health and Observability

Secure-access health is domain-specific.

Remote-access examples:

```text
Policy Engine
Policy Administrator
Policy Enforcement Point
identity provider
MFA dependency
posture dependency
WireGuard listener/peer/session state
Access Session counts/expiry
revocation processing
```

Site-to-site examples:

```text
L2TPv3 tunnel state
IPsec SA state
peer authentication
rekey state
endpoint/WAN reachability
bridge/VLAN/route integration
packet/drop/error state
```

A healthy WireGuard listener or IPsec SA does not prove authorization or application success.

External monitoring consumes Stronghold state and does not become authority for access decisions.

## Failure-State Discipline

Potential future states/reasons may include concepts such as:

```text
AUTHENTICATION_FAILED
MFA_FAILED
DEVICE_NOT_AUTHORIZED
POSTURE_FAILED
RESOURCE_NOT_AUTHORIZED
ACCESS_SESSION_EXPIRED
ACCESS_SESSION_REVOKED
WIREGUARD_PEER_UNAVAILABLE
PE_UNAVAILABLE
PA_UNAVAILABLE
PEP_PROGRAMMING_FAILED
SITE_TUNNEL_DOWN
IPSEC_NEGOTIATION_FAILED
L2TPV3_SESSION_FAILED
REKEY_FAILED
TUNNEL_PATH_UNAVAILABLE
```

Exact names remain implementation work.

Stronghold should record what it can actually establish rather than collapsing these into generic `VPN_DOWN` or `ACCESS_FAILED` states.

## Resource Priority

Secure-access control work must not undermine Stronghold's capture-first resource model.

Remote-access control-plane evaluation, tunnel housekeeping, rekey work, posture processing, and site-tunnel management must be bounded and qualified.

Established production forwarding and essential security-control work may eventually require explicit reserved capacity, but no future secure-access subsystem may silently hide capture loss or make authoritative PCAP durability appear healthy when it is not.

## Security Boundaries

Stronghold must not silently weaken secure transport when configuration or trust fails.

Examples:

```text
WireGuard unavailable
!=
permission to fall back to plaintext remote access

IPsec negotiation failed
!=
permission to run L2TPv3 unprotected

identity provider unavailable
!=
user automatically authorized

posture unavailable
!=
posture passed

revocation processing delayed
!=
revocation completed
```

Any later fail-open/fail-closed behavior must be explicit, scoped, and journaled.

## Qualification Requirements

Before zero-trust remote access is promoted into an implementation phase, freeze and validate at minimum:

```text
NIST PE / PA / PEP responsibility boundaries
Access Session schema/lifecycle
user/device identity model
MFA model
posture model and truth boundaries
Stronghold Access Agent platform/enrollment/update model
WireGuard key/session lifecycle
WireGuard AllowedIPs vs Stronghold resource-policy boundary
tunnel address allocation
resource authorization model
revocation/reauthorization behavior
control-plane outage behavior
PE/PA/PEP performance and scaling
HA behavior
capture/journal representation
Hunter correlation
management/API/RBAC
health/alerting
```

Before site-to-site / branch-office VPN is promoted into an implementation phase, freeze and validate at minimum:

```text
L2TPv3 profile and implementation
IPsec/IKE profile
peer/site authentication model
site/tunnel identity
bridge/VLAN/zone integration
routing interaction where applicable
MTU/fragmentation behavior
multi-WAN interaction
HA/failover/rekey behavior
packet-observation representation
journal schema/events
management/API/RBAC
health/alerting
interoperability/support matrix
performance/PPS/throughput qualification
```

## Truth Separations

```text
WireGuard peer authenticated       != user authenticated
user authenticated                 != device authorized
device authorized                  != resource authorized
tunnel established                 != resource authorized
resource authorized                != connection succeeded
AllowedIPs configured              != Stronghold resource authorization
Access Session created             != application connection succeeded
REVOKE issued                      != revocation fully enforced
L2TPv3 tunnel established          != site authorized
IPsec SA established               != traffic authorized
remote network reachable           != remote network permitted
tunnel healthy                     != application connection succeeded
configured tunnel                  != operational tunnel
outer encrypted packet observed    != inner application authorized
```

## Scope

This document defines future architecture only.

**Zero-trust remote-access and site-to-site / branch-office secure-access implementation remain deferred.**

Neither capability is part of Phase 0, an early dataplane performance claim, or a prerequisite for the current AF_XDP capture foundation.
