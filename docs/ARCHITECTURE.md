# Stronghold Architecture

## Purpose

This document records the current high-level architecture for the complete Stronghold system while preserving the capture-first engineering sequence already established for implementation.

Stronghold is a two-appliance network security system:

- **Stronghold FW** protects and observes the live network; and
- **Stronghold Net-Hunter** receives, verifies, preserves, processes, and exposes historical network activity for hunt/query use.

The complete product is organized around five responsibilities:

```text
OBSERVE
   ↓
RECORD
   ↓
ENFORCE
   ↓
REMEMBER
   ↓
HUNT
```

## Governing Principles

### Capture first

> **Capture first. Never sacrifice observation for secondary work.**

Stronghold must prioritize receiving packets and durably writing active capture data over transfer, compression, indexing, analytics, hunt activity, and other background work.

Net-Hunter services must never become a runtime dependency for forwarding or capture on Stronghold FW.

### Authorization before normal forwarding work

> **A packet does not earn routing, NAT, or deeper forwarding work merely because it is technically routable. Stronghold observes it first, then requires policy authorization before normal forwarding work proceeds.**

Stronghold should reject unauthorized traffic as early and cheaply as practical while preserving the authoritative ingress observation.

> **Denied traffic should be cheap to reject, but never invisible.**

### Explicit path permission

> **Path availability is not path permission.**

A WAN, route, interface, or alternate path is not eligible merely because Stronghold can technically reach it. Administrative policy determines which paths are permitted for a destination/service.

### Journal, do not casually log

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

Administrative actions, system/health state, traffic decisions, trust/identity changes, clock state, and Net-Hunter processing activity belong to separate append-oriented journal domains. Corrections and superseding state create new entries rather than silently rewriting prior history.

## Capture Invariant #1 — Wireshark-Class Interface Visibility

Stronghold's first capture requirement is:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

The authoritative observation point is the configured supported physical interface, not a higher-level assumption about what Linux routes, bridges, or what a firewall rule understands.

A supported Stronghold capture path should provide the same class of visibility expected from Wireshark/dumpcap operating on the same supported physical interface under the same conditions.

When presented to the interface, capture scope includes ordinary IPv4/IPv6 traffic as well as Layer-2 and control-plane traffic such as:

```text
ARP
DHCP / BOOTP
DHCPv6
CDP
LLDP
STP / RSTP / MSTP
LACP
802.1X / EAPOL
OSPFv2 / OSPFv3
VRRP
IGMP
IPv6 NDP
802.1Q VLAN traffic
unknown EtherTypes
unknown IP protocols
vendor-specific frames
malformed traffic
```

Protocol recognition is not a prerequisite for capture.

```text
presented to supported interface
        ↓
record frame / packet
        ↓
preserve authoritative bytes and capture metadata
        ↓
catalog what Stronghold can safely establish
        ↓
decode / enrich later when supported
```

Stronghold must not claim visibility into a frame that the upstream topology, NIC hardware, hardware filtering/offload behavior, or driver did not present to the supported capture path. "On the wire" and "observable by this NIC/capture path" are not always identical; Stronghold reports that distinction truthfully.

NIC offloads and driver behavior that can alter the userspace representation of traffic, including VLAN metadata handling and packet aggregation, must be explicitly evaluated during hardware qualification and capture testing.

## System Topology

```text
                         MANAGEMENT NETWORK
                       ┌──────────┴──────────┐
                       │                     │
                       ▼                     ▼
               Stronghold FW          Stronghold Net-Hunter
                   MGMT                      MGMT


   PRODUCTION NETWORK
          │
          ▼
 ┌────────────────────────────┐
 │       STRONGHOLD FW        │
 │                            │
 │ Observe                    │
 │ Record                     │
 │ Authorize                  │
 │ Bridge / Route             │
 │ Enforce                    │
 │ Journal                    │
 │                            │
 │ Arch Linux                 │
 │ OS SSD / Btrfs             │
 │ NVMe PCAP HOT / XFS        │
 │ local SSD WARM / XFS       │
 └─────────────┬──────────────┘
               │
               │ dedicated Stronghold
               │ history network
               │ 10 / 25 / 40 GbE
               │ mTLS + explicit peer auth
               ▼
 ┌────────────────────────────┐
 │  STRONGHOLD NET-HUNTER     │
 │                            │
 │ Receive                    │
 │ Verify                     │
 │ Preserve                   │
 │ Process                    │
 │ Correlate                  │
 │ Hunt                       │
 │ Query                      │
 │ Export                     │
 │ Journal                    │
 │                            │
 │ FreeBSD                    │
 │ ZFS                        │
 │ Jails                      │
 │ NVMe / SAS SSD / SAS HBA   │
 └────────────────────────────┘
```

## Stronghold FW

Stronghold FW is the live network appliance. It owns the present-tense responsibilities of observing, recording, authorizing, bridging or routing, enforcing, journaling its decisions/state, and maintaining enough local history capacity to survive temporary Net-Hunter unavailability.

### Platform direction

```text
Platform:          Arch Linux
Architecture:      x86_64 / amd64
Administration:    CLI first
NICs:              physical PCIe Ethernet with qualified Linux drivers
Capture:           continuous full-packet capture
Initial source:    AF_PACKET / TPACKET_V3
Format:            PCAPNG
OS storage:        dedicated local SSD / Btrfs direction
PCAP HOT storage:  NVMe / XFS direction
PCAP WARM storage: local SSD / XFS direction
```

### Dataplane targets

```text
1 GbE   current target
10 GbE  primary performance target
40 GbE  future hardware target only
```

Stronghold must not claim validated 10 GbE or 40 GbE operation merely because the architecture anticipates those rates. Performance claims require representative hardware validation under the defined capture and enforcement workload.

USB Ethernet is not part of the supported/reference architecture.

## FW Networking Model

Stronghold FW supports multiple forwarding models. The appliance is not forced into one global Layer-2 or Layer-3 mode.

### Layer-2 transparent bridge

Stronghold may bridge Ethernet/VLAN traffic between configured interfaces or bridge members without becoming the IP default gateway for the bridged network.

```text
Switch / network
       │
       ▼
Stronghold FW
  L2 bridge
  capture
  policy
       │
       ▼
Switch / network
```

### Layer-3 routed firewall

Stronghold may terminate IPv4/IPv6 networks and route between them while applying stateful firewall policy.

```text
Network A
   │
   ▼
Stronghold FW
 route + policy
   │
   ▼
Network B
```

### Router on a stick

Stronghold may terminate multiple 802.1Q VLANs on one physical trunk and route/firewall between them.

```text
               802.1Q trunk
                    │
                    ▼
             Stronghold FW
          ┌─────────┼─────────┐
          │         │         │
       VLAN 10   VLAN 20   VLAN 30
        USERS    SERVERS      DMZ
```

The same physical trunk may carry traffic that enters in one VLAN context and exits through another VLAN context after Stronghold routing and policy enforcement.

### Hybrid deployment

Stronghold may combine Layer-2 bridging and Layer-3 routing on the same appliance where different interfaces, VLANs, or bridge domains require different behavior.

```text
WAN1             Layer 3 routed
LAN-TRUNK        Layer 3 / router on a stick
TRANSIT-A/B      Layer 2 transparent bridge
MGMT             management only
HISTORY          Net-Hunter history only
```

A global appliance mode must not force unrelated interfaces into the same forwarding behavior.

## FW Network Objects

Stronghold configuration should use explicit first-class objects rather than requiring administrators to encode network meaning into Linux interface names or raw nftables statements.

Current object families expected by the architecture include:

```text
physical interface
logical interface
VLAN
bridge domain
zone
host
network
address group
service
service group
FQDN
FQDN group
route
security policy
NAT policy
WAN preference policy
```

Exact schemas remain to be designed.

### VLAN object

A VLAN is a first-class Stronghold object.

At minimum the system must preserve:

```text
Stronghold VLAN identity/name
802.1Q VLAN ID
description/context
associated parent/trunk or bridge/routed use where configured
```

A VLAN object and a logical routed interface are related but not identical concepts. A VLAN may participate in a Layer-2 bridge domain or may be terminated by a Layer-3 logical interface.

Records must preserve both the human Stronghold object identity and the numeric VLAN ID.

### Bridge domain

A bridge domain represents configured Layer-2 forwarding context. It may associate interfaces and/or VLAN context for transparent forwarding and Layer-2 policy.

### Logical interface

A logical interface represents a Stronghold Layer-3 termination point, including VLAN subinterfaces where applicable. It may carry IPv4/IPv6 addressing and participate in routing and policy.

## Zones and Interface Roles

Zones represent security boundaries. Interfaces and VLANs represent actual network attachment.

One routed logical interface belongs to one security zone. A zone may contain multiple logical interfaces or VLANs.

Bridge-domain membership and security-zone membership are distinct concepts; a transparent deployment may use different zones on different sides of a Layer-2 bridge.

Current interface-role model:

```text
PRODUCTION
WAN
MANAGEMENT
HISTORY
```

### PRODUCTION

Carries ordinary bridged or routed network traffic according to policy.

### WAN

Carries production traffic and may participate in WAN health, route preference, NAT identity, scheduled preference, and adaptive path selection.

### MANAGEMENT

Exists for appliance administration and approved management-plane services. It is not an ordinary production transit path.

### HISTORY

Exists for Stronghold FW ↔ Net-Hunter history/control transfer. It is not an ordinary production transit path and must not become a default or alternate production route.

Configuration validation must reject combinations that violate these role boundaries, including a production default route through HISTORY or placing MANAGEMENT into a production bridge domain.

## FQDN Policy Direction

Stronghold should distinguish DNS-backed FQDN policy from directly observed Layer-7 hostname identity.

### DNS-backed FQDN policy

Stronghold may resolve configured names and maintain TTL-aware destination/source address sets for firewall policy.

This proves only that an address was associated with the configured DNS name according to the observed/resolved DNS state. It does not prove that a particular application connection actually used that hostname.

### Layer-7 observed FQDN policy

Future Layer-7 FQDN policy should correlate directly observed application-layer hostname information where available, such as TLS SNI, HTTP Host, DNS history, or other protocol-specific names.

Records must preserve how the hostname was established:

```text
observed hostname: service.example
source: TLS SNI
```

versus:

```text
associated hostname: service.example
source: DNS correlation
```

versus:

```text
hostname: not_observed
```

Stronghold must not convert indirect DNS association into a claim of directly observed Layer-7 hostname identity.

## Policy and Authorization Model

### Default deny

Stronghold is explicit-allow and default-deny for Layer-2 forwarding, Layer-3/4 forwarding, and traffic destined to the appliance itself.

The built-in final disposition for unmatched traffic is DROP.

A firewall DROP or REJECT does not suppress authoritative ingress capture.

### Rule evaluation

Rules are evaluated by their **current rule position**, lowest integer to highest integer.

The first matching rule determines the policy action for that policy domain.

Explicit deny rules may be placed early so known-unwanted traffic can be rejected before unnecessary downstream work.

Rule positions are dense mutable integers:

```text
1
2
3
...
N
```

Inserting a rule at an occupied position pushes the existing rule and all following rules down one position while preserving their relative order.

Moving a rule similarly shifts the affected range while retaining a dense ordered list.

### Policy ID

A Policy ID is a stable reference identity only.

It does not determine evaluation order, rule position, priority, or action.

Historical decision records must preserve at least:

```text
policy_id
policy_name_at_decision_time
rule_position_at_decision_time
configuration_generation
action
```

This lets Net-Hunter reconstruct why traffic was allowed or denied under the exact policy ordering that existed at that time.

### Authorization-first packet processing

The Stronghold semantic processing model is:

```text
PACKET / FRAME ARRIVES
        │
        ├──────────────► CAPTURE / OBSERVE
        │
        ▼
EARLY INGRESS AUTHORIZATION
        │
        ├── explicit DENY ───────────────► DROP
        ├── no permitting match ─────────► DROP
        └── authorized to proceed
                    │
                    ▼
               ROUTING
                    │
                    ▼
        NAT / STATE / REQUIRED
           DEEPER PROCESSING
                    │
                    ▼
              FINAL EGRESS
```

The existence of a viable route or NAT rule never grants permission.

Stronghold should implement the authorization gate as early as practical on the qualified Linux dataplane. Exact nftables/netfilter hook ordering remains an implementation-contract decision and must not be frozen here in a way that conflicts with required kernel semantics.

### Layer-7 deferred final decision

For policy that requires Layer-7 facts, an early authorization may mean only **authorized to continue processing**.

That state permits the packet/session to consume the processing required to establish the final policy condition. It does not mean that Stronghold has already granted final forwarding permission.

Stronghold must not claim a Layer-7 allow before the required application-layer fact has been established.

## Routing Architecture

Only traffic that has passed the applicable authorization gate enters normal Stronghold routing.

Routing answers **where authorized traffic should go**. Routing never answers **whether traffic should be allowed**.

### Route selection

Stronghold route selection follows:

```text
1. longest-prefix match
2. for equal-prefix candidates, route-source preference:
       STATIC
       CONNECTED
       DEFAULT
       DYNAMIC
3. health / availability
4. metric
5. deterministic tie-break
```

A more-specific prefix always wins over a less-specific prefix.

For destination `10.12.12.3`, for example:

```text
10.12.12.3/32
```

wins over:

```text
10.12.12.0/24
```

regardless of route-source preference. Route-source preference applies only among candidates for the same prefix specificity.

### Initial route types

Initial routing direction includes:

```text
connected routes
static routes
default routes
IPv4
IPv6
```

Dynamic routing remains future work unless explicitly brought into scope.

### Route failure

An explicit security allow does not guarantee forwarding success.

```text
Policy: ALLOW
Route:  not_found
Final disposition: DROP
Reason: NO_ROUTE
```

Routing failure must remain distinguishable from firewall-policy denial.

### VRF direction

VRF is an anticipated future capability, not an initial Stronghold requirement.

Initial routing operates in one default routing domain. Interface, route, policy, and record models should preserve a routing-domain field/boundary so future VRF support does not require a redesign.

VRF-aware routing, overlapping address space, and controlled inter-VRF policy are future capabilities.

## Multi-WAN Architecture

Stronghold uses policy-defined WAN preference rather than generic packet-level or flow-spraying load balancing.

### Explicit WAN eligibility

A destination/service policy defines which WANs are allowed to carry that traffic.

Example:

```text
GOOGLE

WAN1
    allowed
    preferred

WAN2
    allowed
    alternate

WAN3
    not_allowed
```

Stronghold may choose only from the explicitly allowed WAN set.

A physically available WAN is not automatically an eligible alternate.

### Preference modes

The architecture anticipates these policy behaviors:

```text
FIXED
    Use the configured WAN only.

PRIMARY/FALLBACK
    Prefer the configured WAN and use an explicitly allowed alternate
    only when the preferred path becomes unavailable.

SCHEDULED
    Change configured preference according to an approved schedule.

ADAPTIVE
    Prefer the configured WAN, but select an explicitly allowed alternate
    for new flows when destination-specific path quality degrades.
```

Scheduled and adaptive preference may be combined.

### Adaptive path quality

Adaptive WAN selection may consider destination-specific measurements such as:

```text
reachability
round-trip time
packet loss
jitter
```

Later measurements may include DNS or application-specific probes when justified.

Thresholds and hysteresis are required so transient measurement noise does not cause rapid path flapping.

A degradation for one destination/service must not automatically mark the entire WAN unusable for unrelated traffic.

### Existing sessions

Established sessions normally remain bound to their established WAN/NAT identity. Adaptive preference changes apply to new eligible flows unless the existing path fails.

Stronghold must not claim seamless migration of a session across WANs when the external identity changes and the protocol/session cannot actually survive that move.

### WAN decision history

The Traffic Decision Journal should preserve enough state to explain the choice, including:

```text
WAN preference policy ID
configured preferred WAN
explicitly eligible WAN set
selected WAN
selection reason
measured path state where applicable
configuration generation
associated route
associated NAT decision
```

## NAT Architecture

NAT is a separate policy domain from security authorization.

The existence of a NAT policy never grants permission to forward traffic.

Current intended NAT capabilities include:

```text
masquerade
static SNAT
DNAT / port forwarding
static 1:1 NAT
NAT exemption / no-NAT
```

Original and translated addressing must remain separately recorded so Net-Hunter can reconstruct the complete historical flow.

NAT must remain tied to the selected WAN/route and state as applicable. Stronghold must not move an established NAT-dependent session between unrelated WAN identities and claim the session was preserved when it was not.

Exact IPv6 translation features such as NAT66/NPTv6 remain future deliberate decisions rather than assumptions copied from IPv4 behavior.

## Configuration Management

Stronghold owns its appliance configuration rather than requiring administrators to edit unrelated Linux files directly.

### Transactional model

```text
RUNNING CONFIGURATION
        │
        ├── copy / edit
        ▼
CANDIDATE CONFIGURATION
        │
        ├── validate
        ├── show diff
        ▼
COMMIT
        │
        ▼
NEW CONFIGURATION GENERATION
```

Candidate changes do not affect live traffic until committed.

### Full-system validation

Validation must include syntax and semantic consistency across the complete resulting configuration.

Examples include:

```text
duplicate or invalid addressing
invalid VLAN relationship
missing referenced object
policy referencing deleted objects/zones
invalid NAT/WAN relationship
unsafe MANAGEMENT/HISTORY role use
invalid next hop
invalid route relationship
rule-position consistency
configuration that violates hard architecture boundaries
```

Stronghold should show the effect of rule insertion/reordering before commit rather than hiding the resulting list change.

### Configuration generations

Each successful commit creates a monotonically advancing configuration generation representing the complete effective configuration.

The generation may include, as applicable:

```text
interfaces
VLANs
bridge domains
zones
routes
WAN preference
NAT
security policy
objects
management settings
Hunter settings
identity/trust settings where applicable
```

Runtime decision records reference the applicable configuration generation.

Policy IDs and other stable object IDs retain identity across generations even when mutable ordering or configuration values change.

### Coordinated activation

Stronghold should prepare and validate required native Linux state before activation and then apply the new generation through the narrowest coordinated/atomic transition practical for the underlying kernel and system components.

Where literal whole-system atomicity is technically impossible, implementation contracts must state and test the real transition boundaries rather than pretending they do not exist.

### Commit-confirmed protection

Management-affecting or otherwise high-risk remote changes should support commit-confirmed behavior.

```text
apply candidate generation
        ↓
confirmation required
        │
        ├── confirmed → generation remains active
        └── not confirmed → automatic rollback
```

The local physical console remains an appliance-recovery path when network management has been broken.

### Rollback

Rollback restores the content of an earlier known-good generation by creating a **new** generation.

Historical generation identity does not move backward and previously committed history is not rewritten.

### Net-Hunter configuration backup

Successful committed generations are intended to be serialized, integrity-protected according to the eventual contract, and preserved as versioned history in the isolated Net-Hunter FW Configuration Backup Jail.

This allows historical packet/decision records to be correlated with the exact Stronghold configuration generation active at the time.

Failed validation, commit, activation, and rollback attempts are Administrative/System Journal events and must not disappear simply because a later retry succeeds.

## Administrative Identity, Authentication, and Authorization

Stronghold separates three concerns:

```text
WHO ARE YOU?             authentication
WHAT MAY YOU DO?         authorization
WHAT DID YOU DO?         journaled administrative history
```

### Local appliance identity

Both Stronghold FW and Net-Hunter require a protected local administrative recovery path.

Local appliance identity exists for installation, physical-console recovery, break-glass, and external-authentication failure. It must not be treated as an invisible automatic fallback that silently weakens remote authentication.

Break-glass/local recovery use is a high-value Administrative Journal event.

External identity-service failure must not stop packet capture, routing, NAT, firewall enforcement, or local recording/journaling.

### Remote authentication sources

The architecture anticipates normal remote authentication through:

```text
Active Directory via LDAPS only
RADIUS
TACACS+
```

#### Active Directory rule

> **Stronghold Active Directory authentication uses LDAPS only. Plaintext LDAP authentication is not supported and there is no automatic LDAPS→LDAP downgrade.**

Stronghold validates the LDAPS server certificate according to the configured trust model, including the applicable issuing trust, certificate validity, and server identity requirements.

LDAPS validation failure is an authentication-service failure, not permission to try insecure LDAP.

Multiple configured domain controllers/LDAPS endpoints should be supported with explicit health/failover behavior.

AD group membership may map to Stronghold-defined roles. Active Directory supplies identity/group facts; Stronghold owns the meaning and permission set of each Stronghold role.

### MFA

Stronghold should support MFA for remote privileged administration, either through Stronghold-managed mechanisms such as TOTP or through an external authentication source that can provide a trustworthy assurance result.

Authentication history should distinguish ordinary authenticated state from MFA-verified state when Stronghold can establish that fact.

### Role-based authorization

The architecture uses role-based authorization and keeps important privileges distinct.

FW-oriented permissions include concepts such as:

```text
configuration.view
configuration.edit_candidate
configuration.validate
configuration.commit
configuration.commit_confirmed
configuration.rollback
network.admin
security_policy.admin
system.admin
```

Net-Hunter permissions include concepts such as:

```text
history.hunt
history.view
history.export
journal.audit_view
system.admin
```

Viewing an investigation is not equivalent to being allowed to export raw or derived packet history.

Candidate-edit and commit authority are separate permissions so later separation-of-duties or two-person workflows do not require an authorization redesign.

### FW and Hunter administration remain separate

> **Net-Hunter access does not imply Stronghold FW administration authority.**

The normal hunt UI must not become an indirect control plane for changing firewall policy/configuration.

If centralized FW administration is later built, it requires a separate privileged control-plane design rather than inheriting trust from the read-only hunt interface.

### Management-plane ingress

Remote administration normally enters through the `MANAGEMENT` role/interface and may be restricted to explicitly configured management source hosts/networks.

The `HISTORY` interface is not a normal interactive administration path.

A valid credential presented from an unauthorized management source does not imply permission to consume the management authentication plane.

### Administrative sessions

Administrative sessions should retain enough journal context to establish:

```text
session identity
user identity
authentication source
MFA state where known
source address/interface
role set
login time
logout/termination reason
```

Sensitive operations may later require reauthentication/MFA confirmation; exact operation classes remain to be frozen.

## Appliance Identity, PKI, and FW↔Hunter Trust

### Stable appliance identity

Each Stronghold appliance has a stable Stronghold Appliance ID independent of:

```text
hostname
IP address
current certificate
management address
```

Certificate rotation therefore changes a credential, not appliance identity.

Packet/history lineage is tied to the originating Appliance ID rather than treating a transient IP address as identity.

### Stronghold-specific trust direction

The preferred architecture is a Stronghold-specific appliance trust hierarchy, separate from ordinary AD user/machine PKI.

A possible direction is separate FW and Hunter issuing authority under a Stronghold trust root, but exact CA topology, offline-root handling, enrollment, renewal, and recovery mechanics remain to be frozen.

### History-link mTLS

Stronghold FW and Net-Hunter mutually authenticate over the dedicated history network using mTLS.

No plaintext fallback is permitted for Stronghold history transport.

Management/UI certificates and history-transfer certificates are separate purposes; one certificate should not automatically be reused for every Stronghold trust role.

### Certificate validity does not equal authorization

A cryptographically valid Stronghold appliance certificate establishes identity. It does **not** automatically authorize that FW to use every Net-Hunter.

Hunter maintains explicit authorized peer relationships.

Conceptually:

```text
certificate valid
        +
appliance explicitly authorized
        +
required transfer role permitted
        ↓
TRANSFER ALLOWED
```

A valid but unauthorized peer is denied.

### Pairing/enrollment

Joining a FW to a Hunter is a deliberate administrative act. Discovery alone must not create trust.

The relationship should preserve:

```text
FW Appliance ID
Hunter Appliance ID
presented/accepted certificate identity
authorization state
relationship state
time/order of authorization
responsible administrative identity/process
```

### Rotation, expiry, and revocation

Certificate rotation preserves the Appliance ID and becomes a Trust/Identity Journal event.

Certificate expiry, revocation, trust failure, or peer-authorization revocation blocks history transfer and creates explicit degraded/backlog state.

These failures do not stop the FW dataplane while local resources remain able to operate.

Stronghold should support explicit peer revocation independent of waiting for an external online revocation service.

CRL/OCSP or other PKI revocation information may augment this design, but Stronghold should avoid introducing an unnecessary public-Internet dependency into the dedicated history network.

### TLS does not replace object integrity

A successful TLS/mTLS session protects and authenticates transport. It does not prove that a capture segment is complete, durable, or matches its Stronghold identity/hash.

Hunter must still independently verify the transferred object before acknowledging committed receipt.

## Time, Clock Authority, and Timestamp Truthfulness

### UTC storage

Stronghold stores authoritative wall-clock timestamps in UTC.

Local timezone selection is a display concern only. Mixed local-time storage must not become part of the historical contract.

### Wall clock versus ordering

Stronghold preserves ordering independently of wall-clock correctness.

Critical journals maintain an advancing local sequence/ordering context. Monotonic time should be used where appropriate for elapsed-time and ordering calculations that must not be broken by wall-clock adjustment.

A wall-clock correction must never silently reverse the causal order of journal entries.

### Time sources

Initial time synchronization should support multiple explicitly configured NTP sources.

The architecture leaves room for:

```text
NTS
PTP
NIC/hardware timestamping
```

where later requirements and hardware qualification justify them.

None is assumed merely because a timestamp format supports fine resolution.

### Clock/source state

The appliance should expose truthful clock states such as:

```text
SYNCHRONIZED
HOLDOVER
UNSYNCHRONIZED
CLOCK_FAULT
```

Individual sources may similarly be represented as healthy, degraded, unreachable, or rejected.

Stronghold should not blindly accept one wildly disagreeing time source when other trusted sources establish that it is inconsistent.

### Clock adjustments are history

Clock corrections, significant offset changes, source failures, changes into/out of holdover, and other meaningful clock-state transitions belong in the Time/Clock Journal.

If time moves backward, journal sequence/order still advances.

### Timestamp accuracy versus resolution

> **Timestamp precision must never be presented as timestamp accuracy.**

A timestamp stored to nanosecond resolution is not a claim that the appliance knew the real event time within one nanosecond.

The capture/record architecture should preserve timestamp-source and clock-confidence information where necessary to make historical correlation truthful.

### Multiple-FW correlation

Net-Hunter may correlate traffic from multiple Stronghold FW appliances. Hunter must preserve each FW's original timestamp and relevant clock state rather than assuming all source clocks had identical accuracy.

A source that was unsynchronized or significantly offset must not be represented as providing sub-millisecond cross-appliance ordering certainty.

### FW time versus Hunter time

Hunter preserves separate time facts, including where applicable:

```text
FW observation/origin time
Hunter receipt time
Hunter verification time
Hunter commit/processing time
```

Hunter does not rewrite the FW's original timestamp to make it agree with Hunter.

### Time-source failure behavior

Loss of external time synchronization must not stop:

```text
capture
routing
NAT
firewall enforcement
journal creation
```

The appliance instead enters an explicit degraded clock-confidence state.

Operations whose security semantics inherently depend on valid time, such as certificate validation, must report their actual failure rather than weakening validation silently.

## Capture Path

```text
physical NIC
 │
 ▼
AF_PACKET / TPACKET_V3
 │
 ▼
RAM ring / capture buffers
 │
 ▼
NVMe XFS HOT capture tier
 │
 ▼
closed capture segment
```

RAM is transient buffering only. A packet is not considered durably captured merely because it reached RAM.

The authoritative configured capture stream is local-first. Downstream processing, retention, transfer, bridging, routing, or policy must not redefine what Stronghold originally observed.

## Observation Versus Forwarding Disposition

Stronghold must preserve the distinction between a packet/frame being observed at a physical interface and what Stronghold subsequently does with that traffic.

Traffic history may therefore preserve distinct facts such as:

```text
physical ingress interface
ingress VLAN
bridge domain or logical interface
source zone / destination zone
authorization decision
Layer-2 forwarding decision
Layer-3 route decision
firewall decision
NAT decision
WAN preference/selection decision
physical/logical egress interface
egress VLAN
configuration generation
associated authoritative capture segment
```

This is especially important for router-on-a-stick traffic, where traffic may enter and leave the same physical interface under different VLAN contexts.

A firewall DROP/REJECT decision must not by itself suppress the authoritative ingress capture.

## Packet Authority and Journal Architecture

### Packet authority

Raw PCAPNG is authoritative for what network traffic Stronghold observed.

Segment identity, capture provenance, packet/byte/drop accounting, and integrity metadata establish packet-history lineage.

A missing decoded/derived record never proves that the underlying packet did not exist.

### Journals are separate domains

Stronghold does not collapse operational history into one general-purpose logfile.

The architecture defines separate journal domains with different authority and use.

#### Administrative Journal

Covers administrative and security-significant human/control-plane actions, including:

```text
login success/failure
logout/session termination
MFA success/failure where known
role/authorization use or change
candidate creation/edit/validation
configuration commit
commit confirmation
automatic/manual rollback
certificate/trust administration
software update
reboot/shutdown
break-glass use
PCAP/report export
retention/destructive administration
```

Candidate creator and committer identities remain separately attributable when different people/processes are involved.

#### System / Health Journal

Covers operational state and failures, including:

```text
interface/link transitions
NIC/driver errors
capture-ring or kernel drops
PCAP writer/storage pressure
HOT/WARM capacity state
filesystem/storage failures
Net-Hunter availability/backlog
route health/withdrawal/restoration
WAN path degradation/restoration
authentication-backend availability
service/jail state
ZFS/storage degradation
```

A later recovery does not erase the earlier failure.

#### Traffic Decision Journal

Covers the forwarding/enforcement decision chain and should preserve enough state to explain what Stronghold did and did not do.

Examples include:

```text
observation context
authorization decision
policy ID/name/position
configuration generation
route decision
WAN eligibility/preference/selection
NAT decision
bridge/routing disposition
final disposition
reason
```

Stronghold explicitly preserves `NOT_PERFORMED` where meaningful.

Example denied early:

```text
authorization: DENY
routing: NOT_PERFORMED
WAN selection: NOT_PERFORMED
NAT: NOT_PERFORMED
final disposition: DROP
```

Example allowed but no route:

```text
authorization: ALLOW
routing: FAILED
reason: NO_ROUTE
NAT: NOT_PERFORMED
final disposition: DROP
```

The exact per-packet versus per-flow/session journal granularity remains an implementation/performance contract. The journal model must not create an unnecessary row-per-packet database requirement when segment/flow authority can represent the needed decision truth more efficiently.

#### Trust / Identity Journal

Covers appliance/authentication trust transitions, including:

```text
appliance enrollment
FW↔Hunter authorization
certificate issuance/installation
certificate rotation
certificate expiry/failure
peer revocation/restoration
unknown/rejected peer attempts
LDAPS/RADIUS/TACACS+ trust/service failures
role/trust-map changes
```

#### Time / Clock Journal

Covers:

```text
clock synchronization state
source selection/health
significant offset changes
clock corrections
holdover entry/exit
unsynchronized/fault transitions
```

#### Hunter Processing Journal

Hunter appends its own history for:

```text
FW history receipt
transfer finalization
independent integrity verification
commit
indexing
correlation/reprocessing
export creation
retention action
processing failure/retry
```

Hunter does not replace the originating FW journal entry with its own processing entry.

### Append-oriented semantics

Committed journal entries are not silently modified in place.

If a fact is later corrected, reinterpreted, superseded, recovered, or invalidated, Stronghold appends a new entry that references the prior state as needed.

Similarly, retention/deletion does not make the historical existence of the removed object disappear. A destructive action creates its own journal entry containing the applicable object identity, authority, time/order, policy/reason, and integrity/lineage information that remains safe and necessary to retain.

### Journal identity and ordering

Exact schemas remain to be frozen, but journal entries should have stable identities and preserve at least:

```text
journal domain / journal identity
entry identity
origin Appliance ID
advancing local sequence/order
origin wall-clock timestamp
clock state/context where relevant
entry type
facts/result
configuration generation where relevant
references to packet/config/peer identities where relevant
```

UUIDv7 remains a candidate for globally unique entry identities, but the exact identity contract is not frozen by this architecture document.

### Journal integrity direction

Critical journals should support append integrity/tamper detection, with hash-linked or otherwise cryptographically verifiable advancement as a strong candidate.

The exact journal finalization/hash/signature contract remains to be frozen before implementation.

PCAP packet-history integrity remains segment-oriented. Stronghold does not require every individual packet to participate in a per-packet cryptographic hash chain merely because journals use append integrity.

### Derived analytical records

Net-Hunter may create derived/enriched analytical records and indexes from authoritative PCAP and source journals.

Derived data is not allowed to overwrite authoritative source history.

Parser/correlation improvements create new/superseding derived interpretation while preserving lineage to the source PCAP/journal facts and, where necessary, the prior derived interpretation.

The UI may present the newest valid interpretation by default without pretending earlier processing never happened.

## Local FW Storage Tiers

Stronghold FW storage is intentionally short-path and capture-focused:

```text
RAM        buffer only
  ↓
NVMe/XFS   HOT authoritative PCAP ingest
  ↓
SSD/XFS    WARM / local outage-backlog capacity
  ↓
Net-Hunter dedicated history network
```

Long-term HDD/RAID history storage belongs to Net-Hunter, not the firewall appliance.

The operating-system filesystem and PCAP filesystem must remain separate so capture-storage exhaustion cannot silently exhaust the root filesystem.

## Net-Hunter Outage Behavior

Net-Hunter availability must not determine whether live capture, bridging, routing, or enforcement continues.

When Net-Hunter is unavailable, Stronghold FW continues to capture and accumulates finalized history locally while capacity permits.

The condition must remain explicit, for example:

```text
Capture:             HEALTHY
Packet loss:         0
Net-Hunter:          UNAVAILABLE
Transfer state:      BACKLOG
Pending history:     known value
Oldest pending:      known timestamp
HOT storage:         known utilization
WARM storage:        known utilization
```

Journal and configuration-history backlog must also remain observable where applicable.

Storage exhaustion and retention behavior require an explicit later contract. Stronghold must not silently discard history or claim continuity that did not occur.

## Capture Segment Lifecycle

The exact implementation contract remains to be frozen, but the intended local FW lifecycle is:

```text
CAPTURING
    ↓
CLOSED
    ↓
SYNCED
    ↓
HASHED
    ↓
CATALOGED
    ↓
VERIFIED
    ↓
TRANSFER_ELIGIBLE
```

Only finalized closed segments are eligible for transfer to Net-Hunter.

A source segment must not be removed merely because a network transfer call returned successfully.

## Dedicated Stronghold History Network

Stronghold FW and Stronghold Net-Hunter communicate through a dedicated physical history interface or network.

Current physical-link direction:

```text
10 GbE / 25 GbE / 40 GbE
```

The history-link speed is independent of the validated firewall dataplane rating. A 25 GbE or 40 GbE history interface is not evidence that the firewall itself is a validated 25 Gb/s or 40 Gb/s forwarding/capture appliance.

The history network is intended for Stronghold history transfer and must not be treated as the ordinary user-management path or production transit path.

## History Transfer and Acknowledgement

Transfer is a verified handoff, not a simple file copy followed by deletion.

```text
FW finalizes segment / source journal batch as applicable
       ↓
FW establishes integrity metadata
       ↓
mTLS-authenticated transfer to authorized Net-Hunter ingest jail
       ↓
Net-Hunter receives complete object
       ↓
Net-Hunter finalizes destination
       ↓
Net-Hunter independently verifies integrity
       ↓
Net-Hunter commits packet/history state
       ↓
Net-Hunter appends Hunter processing journal state
       ↓
Net-Hunter acknowledges verified receipt
       ↓
FW records acknowledgement
```

Source-retention decisions may consider the acknowledged Net-Hunter copy only after applicable verification and commit requirements have completed.

History must retain source lineage, including the originating Stronghold FW Appliance ID, capture-segment identity, and source journal identity/ordering where applicable.

## Stronghold Net-Hunter

Stronghold Net-Hunter holds history and provides the analytical/hunt system.

### Platform direction

```text
Platform:       FreeBSD
Storage:        ZFS
Isolation:      jails
Memory:         large ECC RAM capacity
Fast storage:   NVMe
Warm storage:   SAS SSD as required
Bulk history:   HBA-attached SAS storage under ZFS
```

The FreeBSD host owns infrastructure responsibilities:

```text
physical hardware
HBA and disk visibility
ZFS pools and datasets
NVMe / SAS storage
host networking
PF
jail lifecycle
FreeBSD updates
hardware/storage health
```

Application jails must not receive raw host/storage authority merely because they consume a dataset.

## Net-Hunter Jail Architecture

Net-Hunter currently defines four application jails.

### 1. PCAP Data Ingest Jail

Purpose: safely receive current history from explicitly authorized Stronghold FW appliances.

Responsibilities include:

```text
authenticate/authorize source Stronghold FW over mTLS
receive finalized PCAP segments
receive associated source journals/initial records
verify segment/journal identity
verify transfer completeness
verify integrity requirements
commit received history
append Hunter receive/verify state
acknowledge verified receipt
```

It does not provide hunt/query user access.

### 2. Record Processing Jail

Purpose: turn authoritative packet history and source FW journals/records into searchable historical knowledge.

Responsibilities may include:

```text
flow/session construction
DHCP processing
ARP/NDP processing
DNS processing
CDP/LLDP processing
STP processing
OSPF processing
Layer-2 forwarding correlation
routing/firewall/NAT correlation
WAN-selection correlation
MAC/IP relationships
record enrichment
index creation/maintenance
historical reprocessing when decoders improve
```

This jail may read authoritative PCAP and source journals but must not rewrite authoritative source history as part of normal processing.

### 3. External User Interface Jail

Purpose: expose Stronghold history to authorized users for investigation.

Capabilities may include:

```text
hunt
query
search
timeline
correlation
record/journal viewing
PCAP retrieval
controlled export
reports
history/system status
```

#### External UI read-only invariant

> **The External User Interface may query, interpret, correlate, display, and export Stronghold history, but it may not alter authoritative PCAP, source journals, processed records, indexes, or firewall configuration history.**

The UI jail may receive writable non-authoritative workspace for session state, temporary queries, derived PCAP exports, reports, investigation notes, and download staging.

A user-created note/export does not become authoritative Stronghold source history and does not modify the journal/record it references.

Packet export should be a separately authorizable permission and every export is an Administrative/Hunter Processing Journal event as applicable.

### 4. FW Configuration Backup Jail

Purpose: preserve versioned Stronghold FW configuration backups for controlled recovery and historical correlation.

Allowed access is intentionally narrow:

```text
authorized Stronghold FW appliance
local Net-Hunter administrator
```

Ordinary external UI/hunt users, the record-processing jail, and unrelated application services do not receive access to firewall configuration history.

Configuration history should be versioned/append-oriented rather than represented by one repeatedly overwritten backup object.

The exact backup/restore trust, secret-handling, and authorization contract remains to be designed.

## Net-Hunter Storage Direction

Net-Hunter storage should separate high-IOPS processing/query workloads from large sequential historical-PCAP workloads where practical.

```text
NVMe FAST
    ingest work
    indexes
    active records/journals
    query/scratch
    recent history

SAS SSD WARM
    recent/frequently accessed history
    processing/index workloads where appropriate

HBA → SAS ZFS HISTORY
    authoritative long-term PCAP
    journal/history retention
    historical retrieval
```

ZFS owns redundancy, checksumming, scrubs, and pool-level integrity for Net-Hunter storage. Stronghold's own packet/history identities, hashes, journal advancement, and transfer lineage remain separate application-level truth and must not be replaced by filesystem state alone.

RAIDZ2 is a current candidate for bulk history storage, but exact vdev width/topology remains to be frozen by storage sizing and measurement.

Hardware RAID must not hide bulk history disks from ZFS in the intended architecture.

## Net-Hunter Read/Write Boundaries

| Resource | PCAP Ingest | Record Processing | External UI | FW Config Backup |
| --- | --- | --- | --- | --- |
| Receive FW PCAP | controlled write | no | no | no |
| Receive source FW journals | controlled write | no | no | no |
| Authoritative stored PCAP | commit-controlled | read-only | read-only | no |
| Source FW journals/initial records | receive/write | read-only | read-only | no |
| Processed records/indexes | no | read/write | read-only | no |
| Hunter Processing Journal | append as applicable | append as applicable | read-only | no |
| Hunt/query history | no | produces | read-only | no |
| Derived export workspace | no | as required | writable non-authoritative | no |
| FW configuration backups | no | no | no access | controlled read/write |
| External user access | no | no | yes | no |

Exact dataset mounts, credentials, and OS permissions remain future implementation details, but implementation must not silently weaken these boundaries.

## Resource Priority

On Stronghold FW, intended scheduling priority remains:

```text
1. packet acquisition
2. active PCAP writes to HOT storage
3. segment finalization and minimum integrity work
4. essential observation/decision/journal state
5. local tier movement / backlog handling
6. transfer to Net-Hunter
7. compression where approved
8. deeper indexing / analytics outside the live path
```

Within the live forwarding path, early authorization should reject clearly unauthorized traffic before unnecessary routing, NAT, or deeper processing work whenever the qualified kernel/data path can do so without weakening correctness or capture visibility.

Net-Hunter processing and user queries are isolated on a separate appliance so hunt activity cannot consume FW capture resources.

## Architectural Summary

Stronghold FW owns the present:

```text
observe → record → authorize → bridge/route → enforce → journal → hand off verified history
```

Stronghold Net-Hunter owns the past:

```text
receive → verify → preserve → journal → process → correlate → hunt → query → export
```

The system must preserve the distinction between:

```text
authoritative PCAP packet history
source FW journals
Hunter processing journals
derived analytical records/indexes
runtime policy/routing/NAT/WAN decisions
configuration generations
user-generated notes/views/exports
```

Later implementation phase sequencing remains intentionally unfrozen while the complete product architecture is still being defined.
