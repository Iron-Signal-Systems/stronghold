# Stronghold Architecture

## Purpose

This document records the current high-level architecture for the complete Stronghold system while preserving the capture-first engineering sequence already established for implementation.

Stronghold is a two-appliance network security system:

- **Stronghold FW** protects and observes the live network; and
- **Stronghold Net-Hunter** receives, preserves, processes, and exposes historical network activity for hunt/query use.

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
 │                            │
 │ FreeBSD                    │
 │ ZFS                        │
 │ Jails                      │
 │ NVMe / SAS SSD / SAS HBA   │
 └────────────────────────────┘
```

## Stronghold FW

Stronghold FW is the live network appliance. It owns the present-tense responsibilities of observing, recording, authorizing, bridging or routing, enforcing, and maintaining enough local history capacity to survive temporary Net-Hunter unavailability.

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

### WAN records

Historical records should preserve:

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

Failed validation, commit, activation, and rollback attempts should also produce durable administrative/system records.

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

A traffic record may contain distinct facts such as:

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

Storage exhaustion and retention behavior require an explicit later contract. Stronghold must not silently discard history or claim continuity that did not occur.

## Records and Packet Authority

Raw PCAPNG is the authoritative packet history. Structured records explain, locate, correlate, and make that history usable.

The system should preserve distinct classes of record, including:

```text
packet/capture provenance
physical interface observation
MAC/VLAN context
bridge-domain context
flow/session facts
DHCP/ARP/NDP/DNS and other safely decoded protocol facts
CDP/LLDP/STP/OSPF and other control-plane facts
authorization decisions
Layer-2 forwarding decisions
Layer-3 routing decisions
firewall decisions
NAT decisions
WAN preference/selection decisions
packet-loss/degraded-state facts
configuration generation
system and administrative activity
transfer/verification history
```

The firewall should create initial records that can be safely established while traffic is live.

Net-Hunter may enrich those records and reprocess older PCAP when future decoders or correlation logic improve.

> **Failure to recognize, decode, enrich, or index traffic must never cause the underlying authoritative packet history to be discarded.**

Absence of a decoded record is not proof that the underlying traffic did not exist.

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
FW finalizes segment
       ↓
FW establishes integrity metadata
       ↓
transfer to Net-Hunter ingest jail
       ↓
Net-Hunter receives complete object
       ↓
Net-Hunter finalizes destination
       ↓
Net-Hunter independently verifies integrity
       ↓
Net-Hunter commits packet/history state
       ↓
Net-Hunter acknowledges verified receipt
       ↓
FW records acknowledgement
```

Source-retention decisions may consider the acknowledged Net-Hunter copy only after applicable verification and commit requirements have completed.

History must retain source lineage, including the originating Stronghold FW identity and capture-segment identity.

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

Purpose: safely receive current history from authorized Stronghold FW appliances.

Responsibilities include:

```text
authenticate source Stronghold FW
receive finalized PCAP segments
receive associated initial records
verify segment identity
verify transfer completeness
verify integrity/hash requirements
commit received history
acknowledge verified receipt
```

It does not provide hunt/query user access.

### 2. Record Processing Jail

Purpose: turn authoritative packet history and initial FW records into searchable historical knowledge.

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

This jail may read authoritative PCAP but must not rewrite authoritative packet history as part of normal processing.

### 3. External User Interface Jail

Purpose: expose Stronghold history to authorized users for investigation.

Capabilities may include:

```text
hunt
query
search
timeline
correlation
record viewing
PCAP retrieval
controlled export
reports
history/system status
```

#### External UI read-only invariant

> **The External User Interface may query, interpret, correlate, display, and export Stronghold history, but it may not alter authoritative PCAP, observation records, processed records, indexes, or firewall configuration history.**

The UI jail may receive writable non-authoritative workspace for session state, temporary queries, derived PCAP exports, reports, and download staging.

A user-created export does not become authoritative Stronghold history merely because Stronghold generated it.

### 4. FW Configuration Backup Jail

Purpose: preserve versioned Stronghold FW configuration backups for controlled recovery and historical correlation.

Allowed access is intentionally narrow:

```text
Stronghold FW appliance
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
    active records
    query/scratch
    recent history

SAS SSD WARM
    recent/frequently accessed history
    processing/index workloads where appropriate

HBA → SAS ZFS HISTORY
    authoritative long-term PCAP
    retention
    historical retrieval
```

ZFS owns redundancy, checksumming, scrubs, and pool-level integrity for Net-Hunter storage. Stronghold's own packet/history records, segment identities, hashes, and transfer lineage remain separate application-level truth and must not be replaced by filesystem state alone.

RAIDZ2 is a current candidate for bulk history storage, but exact vdev width/topology remains to be frozen by storage sizing and measurement.

Hardware RAID must not hide bulk history disks from ZFS in the intended architecture.

## Net-Hunter Read/Write Boundaries

| Resource | PCAP Ingest | Record Processing | External UI | FW Config Backup |
| --- | --- | --- | --- | --- |
| Receive FW PCAP | controlled write | no | no | no |
| Authoritative stored PCAP | commit-controlled | read-only | read-only | no |
| Initial FW records | receive/write | read-only | read-only | no |
| Processed records/indexes | no | read/write | read-only | no |
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
4. essential observation/record state
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
observe → record → authorize → bridge/route → enforce → hand off verified history
```

Stronghold Net-Hunter owns the past:

```text
receive → verify → preserve → process → correlate → hunt → query → export
```

The system must preserve the distinction between authoritative packet history, structured records derived from that history, runtime policy/routing/NAT decisions, configuration generations, and user-generated views/exports of that history.

Later implementation phase sequencing remains intentionally unfrozen while the complete product architecture is still being defined.
