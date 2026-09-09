# Stronghold Endpoint Enforcement Architecture

## Purpose

This document defines the current future architecture for a Stronghold-managed endpoint enforcement component, beginning with Windows.

**Endpoint-enforcement implementation remains deferred.** This document establishes architectural boundaries for a later Stronghold Endpoint Agent and does not pull endpoint code, Windows Filtering Platform integration, deep Layer-7 inspection, or endpoint policy distribution into Phase 0.

The governing idea is:

> **Stronghold may enforce policy at the endpoint before unauthorized traffic reaches the network, while Stronghold FW remains the independent network enforcement authority.**

The endpoint component is not a replacement firewall, not an EDR, and not an independent policy authority. It is an additional Stronghold Policy Enforcement Point located close to the originating process.

## Relationship to Stronghold Secure Access

`docs/SECURE-ACCESS.md` defines Stronghold zero-trust remote access using the NIST SP 800-207 Policy Engine / Policy Administrator / Policy Enforcement Point direction with WireGuard as the secure remote-access transport.

Endpoint enforcement extends that model so a managed endpoint may participate as a client-side Policy Enforcement Point while Stronghold FW remains the network-side Policy Enforcement Point.

Conceptually:

```text
                         STRONGHOLD CONTROL PLANE

                   Policy Engine / Policy Administrator
                                │
                    signed policy / session state
                                │
               ┌────────────────┴────────────────┐
               │                                 │
               ▼                                 ▼
      WINDOWS ENDPOINT PEP                 STRONGHOLD FW PEP
      Stronghold Endpoint Agent            network enforcement
      Windows Filtering Platform           capture / journal
               │                                 │
               └──────────── traffic ────────────┘
```

For remote zero-trust access, the relationship may become:

```text
Policy Engine / Policy Administrator
               │
       ┌───────┴────────┐
       ▼                ▼
Endpoint PEP        FW PEP
Windows Agent       Stronghold FW
       │                │
       └── WireGuard ───┘
```

The endpoint PEP may reject traffic before it enters WireGuard or reaches the local network. The FW PEP independently applies normal Stronghold authorization to traffic that does reach it.

## Governing Authorization Boundary

> **Endpoint ALLOW never grants network permission.**

The endpoint and FW enforce at different locations and maintain independent outcomes.

Mandatory separations:

```text
endpoint allow
!=
FW allow

endpoint deny
!=
FW observed deny

endpoint policy installed
!=
FW policy installed

endpoint PEP healthy
!=
FW PEP healthy

application identified
!=
application trusted

device on a VLAN
!=
device authorized for that VLAN/zone

local network
!=
trusted endpoint
```

A packet or connection that survives endpoint enforcement still requires normal Stronghold FW authorization when it reaches the FW.

The endpoint may reduce unnecessary traffic presented to the FW; it must never create a bypass around FW policy.

## Initial Windows Direction

The first endpoint platform direction is Windows.

Conceptually:

```text
Stronghold Endpoint Agent
    Windows service
    Go
        ↓
Windows-native enforcement integration
        ↓
Windows Filtering Platform
        ↓
local endpoint Policy Enforcement Point
```

Stronghold should use native Windows enforcement facilities rather than implement endpoint filtering by repeatedly parsing shell-command output or maintaining a competing user-space packet stack.

The initial direction is connection/application-aware enforcement using Windows Filtering Platform capabilities appropriate to user-mode policy management and connection authorization.

Initial endpoint facts may include:

```text
user / subject identity
device identity
application/process identity
destination IP/resource
destination port/service
protocol
direction
network/zone context
Stronghold Access Session
Policy Generation
authorization state
```

The exact Windows APIs, WFP layer selection, filter ownership, service privileges, installation model, signing requirements, and persistence model must be frozen before implementation.

## Layer-7 Boundary

Stronghold must not call every endpoint connection decision "deep Layer 7 inspection."

The initial endpoint PEP can provide valuable application-aware and resource-aware enforcement without parsing arbitrary application payloads.

For example:

```text
User:        DOMAIN\\John
Device:      LAPTOP-42
Application: mstsc.exe
Resource:    SERVER-17
Service:     TCP/3389
Policy:      Server-Administrators
Decision:    ALLOW
```

or:

```text
User:        DOMAIN\\John
Device:      LAPTOP-42
Application: powershell.exe
Resource:    SQL-PROD
Service:     TCP/1433
Decision:    DENY
```

The second connection may be blocked locally before the traffic traverses the network.

True payload-aware Layer-7 behavior is a separate later capability.

Potential later approaches may include:

```text
selective local application proxy

or

signed, tightly scoped WFP callout-driver architecture
    +
Go control/service layer
```

A kernel-mode DPI/callout driver is **not** the initial Endpoint Agent requirement. It must be justified by measured operational value and receive separate security, signing, compatibility, update, crash, and performance qualification.

The endpoint project must not become an EDR/DPI framework merely because Windows permits deeper interception.

## Local Network Context

The Endpoint Agent may consume Stronghold-provided network context so policy can reflect where the managed endpoint is operating.

The important context is not merely a VLAN number. Stronghold should reason from a combination such as:

```text
Endpoint / Device ID
Stronghold Zone
observed network context
VLAN context where known
Policy Generation
control-plane source identity
```

Conceptually:

```text
Endpoint:           LAPTOP-42
Device certificate: <identity>
Observed IP:        10.42.17.23
Observed zone:      COUNTY-USERS
Observed VLAN:      42
Policy Generation:  1837
```

The FW/network-side observation may be more authoritative for access-VLAN context than the endpoint because an endpoint attached to an untagged switchport may never receive the VLAN tag itself.

Mandatory separation:

```text
endpoint reports VLAN/zone
!=
Stronghold network-side context established
```

Stronghold does not let a workstation self-assert a trusted zone and thereby gain authorization.

## Signed Endpoint Policy Generations

Endpoint policy distribution should follow the same Stronghold configuration discipline used elsewhere.

Conceptually:

```text
Stronghold control authority
        ↓
SIGNED ENDPOINT POLICY GENERATION
        ↓
Endpoint receives candidate
        ↓
verify signature / source authority
        ↓
validate
        ↓
program endpoint enforcement state
        ↓
verify activation
        ↓
activate generation
        ↓
journal result
```

A policy bundle may include or reference:

```text
Endpoint ID / target scope
Policy Generation
zone / network context
valid-from time
expiration / lease time
resource/application/service rules
Stronghold Access Session state where applicable
issuer / signing identity
integrity/signature metadata
```

Exact serialization and signature algorithms remain to be frozen with the Stronghold cryptographic profile.

Policy generations are monotonic within their defined scope. A later generation does not silently become active merely because bytes were received.

Truth separations:

```text
policy received
!=
policy verified

policy verified
!=
policy activated

policy activated
!=
endpoint enforcement healthy

new policy available
!=
old policy safely replaced
```

## Local Operation

When a managed Windows endpoint is on a Stronghold-protected local network, WireGuard is not automatically required merely because the Endpoint Agent is installed.

Conceptually:

```text
LAPTOP-42
    ↓
Endpoint Agent / endpoint PEP
    ↓
local network
    ↓
Stronghold FW / network PEP
    ↓
resource
```

The endpoint may enforce application/user/device/resource rules locally before traffic reaches the FW.

The FW remains authoritative for its own network decision and continues normal physical-interface observation, policy, routing, NAT, WAN, and journaling behavior for traffic actually presented to it.

## Remote / Zero-Trust Operation

When the endpoint is remote or otherwise operating in an untrusted network context, the same Endpoint Agent may participate in Stronghold Zero-Trust Access.

Conceptually:

```text
Endpoint Agent
    ↓
user / device / MFA / posture
    ↓
Stronghold Access Session
    ↓
endpoint PEP policy
    ↓
WireGuard secure transport
    ↓
Stronghold FW PEP
    ↓
resource
```

An Access Session may authorize a narrow tuple such as:

```text
application: mstsc.exe
resource:    SERVER-17
service:     TCP/3389
```

while the endpoint PEP rejects other local applications attempting to use the same secure-access path.

The FW independently validates and enforces the Access Session/resource authorization.

```text
WireGuard available
!=
endpoint application authorized

endpoint application authorized
!=
FW resource authorized
```

## Local Denial Before Network Transmission

One intended operational benefit is to reject unauthorized activity close to the originating process.

Conceptually:

```text
unauthorized process
      ↓
Windows endpoint PEP
      ↓
DENY
```

The traffic does not need to consume WireGuard transport or Stronghold FW dataplane resources when the endpoint has sufficient authoritative policy to reject it locally.

Stronghold must remain truthful about what actually occurred:

```text
endpoint denied before network transmission
```

is an endpoint fact.

It must not be rewritten as:

```text
FW denied the packet
```

if the FW never observed that traffic.

## Endpoint Decision Records

Endpoint decisions should be attributable and correlated with Stronghold policy/session state.

A conceptual endpoint decision record may contain:

```text
Endpoint Decision ID
Endpoint / Device ID
user / subject identity
application/process identity
process-path/signature identity where later approved
source network context
destination/resource
service/protocol
Access Session ID where applicable
Policy Generation
decision
reason
result
UTC time
endpoint clock-confidence state where applicable
control-plane policy source
enforcement health state
```

Exact journal/export schema remains future work.

Potential decisions/results include:

```text
ALLOW
DENY
REVOKED
NOT_PERFORMED
POLICY_UNAVAILABLE
POLICY_EXPIRED
ENFORCEMENT_FAILED
```

The endpoint should not reduce all failures to `BLOCKED` or `VPN_FAILED` when it knows which stage actually failed.

## Endpoint Journaling and Authority

Endpoint enforcement records are distinct from FW Traffic Decision Journal facts.

Conceptually:

```text
ENDPOINT ENFORCEMENT HISTORY
    what the endpoint PEP decided/performed

FW TRAFFIC DECISION JOURNAL
    what Stronghold FW decided/performed
```

The exact long-term authority/integrity model for endpoint-origin records remains to be frozen before implementation.

Any future endpoint journal/signing identity must be purpose-separated from WireGuard peer keys and from FW/Hunter appliance identities.

A later Net-Hunter correlation does not convert an endpoint record into a FW observation.

## Hunter Correlation

Net-Hunter may correlate endpoint decisions with Stronghold Access Sessions, FW journals, and authoritative packet history.

Useful pivots may include:

```text
Endpoint / Device ID
user
application/process
Access Session ID
Policy Generation
resource
service
endpoint decision
FW decision
WireGuard session
related packet segments
```

Example derived timeline:

```text
16:31:02  Endpoint PEP denied powershell.exe → SERVER-17:3389
16:31:14  Endpoint PEP allowed mstsc.exe → SERVER-17:3389
16:31:14  FW PEP authorized Access Session AS-192
16:31:15  RDP connection established
```

Hunter correlation is derived interpretation. It does not rewrite the endpoint source record, FW source journal, or authoritative PCAP.

Net-Hunter remains optional for basic endpoint enforcement unless a later explicit requirement changes that boundary.

## Failure and Holdover Behavior

The Endpoint Agent must not silently disable enforcement merely because the FW/control plane is temporarily unreachable.

Endpoint policy should use explicit validity/lease semantics.

Conceptually:

```text
POLICY CURRENT
POLICY HOLDOVER
POLICY EXPIRED
CONTROL UNAVAILABLE
ENFORCEMENT DEGRADED
ENFORCEMENT FAILED
```

A cached policy remains distinguishable from a freshly validated policy.

Example:

```text
Policy Generation: 1837
Issued:            16:00 UTC
Valid Until:       20:00 UTC
Source:            FW-A / Cluster-1 control authority
Signature:         VALID
State:             POLICY HOLDOVER
```

Exact fail-open/fail-closed behavior must be explicit and scoped by use case. Stronghold must not globally assume either:

```text
control unavailable → allow everything
```

or:

```text
control unavailable → brick workstation networking
```

Different later policies may be required for ordinary local networking, remote zero-trust access, privileged administration, emergency/break-glass access, and expired endpoint policy.

All such behavior requires explicit configuration, state visibility, and journaling.

## Endpoint Health

Endpoint enforcement health is independent from FW health.

Potential dimensions include:

```text
Endpoint Agent service state
policy generation/currentness
policy signature validation
WFP programming state
filter activation/verification
control-channel health
Access Session synchronization
WireGuard integration where applicable
clock state
endpoint identity/certificate state
local resource pressure
update/version state
```

```text
Endpoint Agent running
!=
endpoint PEP healthy
```

```text
policy activated
!=
WFP enforcement verified
```

An endpoint may be degraded while the FW is healthy, or vice versa.

## Security Boundaries

The Endpoint Agent is security-sensitive local software and must follow Stronghold's normal explicit-authority model.

Future implementation must address at minimum:

```text
service identity and Windows privileges
binary/code signing
installer/update signing
configuration/policy signature verification
local secret/key storage
device identity protection
anti-downgrade behavior
rollback behavior
WFP filter ownership and cleanup
service crash/restart behavior
local administrator tampering boundary
policy replay protection
control-channel authentication
time/expiration handling
```

Stronghold must remain truthful about the local-administrator/root-equivalent threat boundary. A sufficiently privileged local attacker may be able to tamper with endpoint software or the operating system; the design must detect/report what can be established rather than claim impossible local tamper immunity.

## Resource Discipline

Endpoint enforcement should be lightweight and bounded.

The initial Go service should not perform full payload inspection merely because a connection can be observed.

Expected early priority:

```text
1. correct local enforcement
2. policy/session validity
3. decision recording
4. control synchronization
5. diagnostics/metrics
6. optional later deeper inspection
```

Endpoint telemetry/control traffic must not become an unbounded stream to Stronghold FW.

Policy updates should be generation/delta oriented where practical rather than pushing a complete rule universe repeatedly for every minor network event.

## Initial Implementation Boundary

The preferred first endpoint implementation slice is:

```text
Windows
    ↓
Go service
    ↓
Stronghold device/policy identity
    ↓
signed endpoint policy generations
    ↓
Windows Filtering Platform connection/application enforcement
    ↓
endpoint decision records
    ↓
local-network + future Zero-Trust Access integration
```

The following are explicitly **not** initial requirements:

```text
kernel-mode DPI engine
general malware detection
EDR behavior
full payload retention
TLS interception on endpoint
generic application sandboxing
memory inspection
process injection
kernel callback collection unrelated to enforcement
```

Those capabilities require separate justification and architecture if ever adopted.

## Later Deep Layer-7 Qualification

If deeper endpoint Layer-7 enforcement is later justified, freeze and validate at minimum:

```text
exact L7 question being solved
why connection/application identity is insufficient
local proxy vs kernel callout-driver architecture
supported protocols
TLS handling
certificate/key boundaries
payload retention boundary
privacy/sensitivity implications
kernel driver signing/update/recovery
Windows version compatibility
crash/BSOD risk
performance/latency
failure behavior
endpoint resource pressure
debug/support artifact exposure
journal/provenance model
```

Deep Layer-7 support must remain selective. It must not turn the Endpoint Agent into an unbounded endpoint packet-inspection product by default.

## Qualification Requirements

Before Endpoint Enforcement is promoted into an implementation phase, freeze and validate at minimum:

```text
Windows versions / editions supported
Go service lifecycle
installer/update/signing model
endpoint identity/enrollment
control-channel authentication
policy bundle schema/signing
policy generation/rollback rules
policy lease/holdover/expiry semantics
WFP integration/layer/filter ownership
application/process identity semantics
user/device identity semantics
zone/VLAN/network-context authority
local vs remote operation
Access Session integration
WireGuard integration
endpoint/FW dual-PEP semantics
endpoint decision record schema
endpoint record integrity/transport
Hunter correlation
health/alerting
local-admin tamper boundary
performance/latency/resource use
upgrade/recovery behavior
uninstall/filter cleanup behavior
```

Exact implementation phase placement remains deferred.

## Truth Separations

```text
endpoint allow                    != FW allow
endpoint deny                     != FW observed deny
endpoint policy installed         != FW policy installed
endpoint PEP healthy              != FW PEP healthy
application identified            != application trusted
device on VLAN                    != device authorized
endpoint VLAN claim               != network-side VLAN context established
local network                     != trusted endpoint
policy received                   != policy verified
policy verified                   != policy activated
policy activated                  != enforcement healthy
cached policy                     != current control-plane policy
WireGuard available               != resource authorized
endpoint application authorized   != FW resource authorized
endpoint record                   != FW journal record
endpoint record                   != authoritative physical-interface observation
Hunter correlation                != original endpoint/FW knowledge
WFP filter programmed             != connection successfully blocked
connection locally denied         != packet observed by FW
deep Layer-7 available            != deep Layer-7 required
```

## Scope

This document defines future architecture only.

Endpoint Enforcement is not part of Phase 0. No endpoint agent, WFP integration, endpoint policy distribution, endpoint journal, endpoint health system, or deep Layer-7 capability is pulled into the current AF_XDP Traffic Observation Foundation without explicit approval.