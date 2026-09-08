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

## Governing Principle

> **Capture first. Never sacrifice observation for secondary work.**

Stronghold must prioritize receiving packets and durably writing active capture data over transfer, compression, indexing, analytics, hunt activity, and other background work.

Net-Hunter services must never become a runtime dependency for forwarding or capture on Stronghold FW.

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

Stronghold FW is the live network appliance. It owns the present-tense responsibilities of observing, recording, bridging or routing, enforcing, and maintaining enough local history capacity to survive temporary Net-Hunter unavailability.

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

Current Stronghold FW dataplane direction:

```text
1 GbE   current target
10 GbE  primary performance target
40 GbE  future hardware target only
```

Stronghold must not claim validated 10 GbE or 40 GbE operation merely because the architecture anticipates those rates. Performance claims require representative hardware validation under the defined capture and enforcement workload.

USB Ethernet is not part of the supported/reference architecture.

## FW Networking Model

Stronghold FW supports multiple forwarding models. The appliance is not forced into one global Layer-2 or Layer-3 mode.

The architectural deployment models are:

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

This mode exists for transparent insertion where Layer-3 addressing and gateway design should remain unchanged.

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

Connected and static routing are the initial intended routing direction. Dynamic routing remains future work unless explicitly brought into scope.

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

The same physical trunk may therefore carry traffic that enters in one VLAN context and exits through another VLAN context after Stronghold routing and policy enforcement.

### Hybrid deployment

Stronghold may combine Layer-2 bridging and Layer-3 routing on the same appliance where different interfaces, VLANs, or bridge domains require different behavior.

Example:

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
```

Exact object schemas and policy syntax remain to be designed.

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

## FQDN Policy Direction

Stronghold should eventually distinguish two different kinds of FQDN knowledge rather than conflating them.

### DNS-backed FQDN policy

Stronghold may resolve configured names and maintain TTL-aware destination/source address sets for firewall policy.

This proves only that an address was associated with the configured DNS name according to the observed/resolved DNS state. It does not prove that a particular application connection actually used that hostname.

### Layer-7 observed FQDN policy

Future Layer-7 FQDN policy should correlate directly observed application-layer hostname information where available, such as TLS SNI, HTTP Host, DNS history, or other protocol-specific names.

Records must preserve how the hostname was established, for example:

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

A traffic record may therefore contain distinct facts such as:

```text
physical ingress interface
ingress VLAN
bridge domain or logical interface
Layer-2 forwarding decision
Layer-3 route decision
firewall decision
NAT decision
physical/logical egress interface
egress VLAN
associated authoritative capture segment
```

This is especially important for router-on-a-stick traffic, where traffic may enter and leave the same physical interface under different VLAN contexts. Stronghold must correlate the disposition without pretending that the physical observations are unrelated traffic.

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

When Net-Hunter is unavailable, Stronghold FW continues to capture and accumulates finalized history locally while capacity permits. The condition must remain explicit, for example:

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
Layer-2 forwarding decisions
Layer-3 routing decisions
firewall decisions
NAT decisions
packet-loss/degraded-state facts
system and administrative activity
transfer/verification history
```

The firewall should create the initial records that can be safely established while the traffic is live.

Net-Hunter may enrich those records and reprocess older PCAP when future decoders or correlation logic improve.

The following invariant applies:

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

The history network is intended for Stronghold history transfer and must not be treated as the ordinary user-management path.

## History Transfer and Acknowledgement

Transfer is a verified handoff, not a simple file copy followed by deletion.

Conceptual sequence:

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

Source-retention decisions may consider the acknowledged Net-Hunter copy only after the applicable verification and commit requirements have completed.

History must retain source lineage, including the originating Stronghold FW identity and capture-segment identity.

## Stronghold Net-Hunter

Stronghold Net-Hunter holds the history and provides the analytical/hunt system.

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

Purpose: preserve versioned Stronghold FW configuration backups for controlled recovery.

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

Conceptually:

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

## Net-Hunter Read/Write Boundaries

The intended trust model is asymmetric:

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

Exact dataset mounts, credentials, and OS permissions remain future implementation details, but later implementation must not silently weaken these boundaries.

## Resource Priority

On Stronghold FW, intended scheduling priority remains:

```text
1. packet acquisition
2. active PCAP writes to HOT storage
3. segment finalization and minimum integrity work
4. essential observation/record state
5. live bridge/route/firewall work required for forwarding
6. local tier movement / backlog handling
7. transfer to Net-Hunter
8. compression where approved
9. deeper indexing / analytics outside the live path
```

Capture and required live forwarding/enforcement are real-time responsibilities. Net-Hunter processing and user queries are isolated on a separate appliance so hunt activity cannot consume FW capture resources.

## Future FW Networking and Enforcement Layers

The following remain intentionally later than the initial capture foundation even though their architecture is now recorded:

- Layer-2 bridge domains and transparent policy;
- IPv4/IPv6 interface configuration;
- VLAN and logical-interface configuration;
- router-on-a-stick operation;
- connected/static routing;
- nftables stateful firewall policy;
- host/network/VLAN/service/FQDN objects and groups;
- NAT;
- DNS-backed FQDN policy;
- Layer-7 observed FQDN policy;
- firewall/routing/bridge correlation with captured traffic; and
- additional security features only when explicitly added to the roadmap.

Stronghold should not implement these early merely because the complete architecture anticipates them.

## Architectural Summary

Stronghold FW owns the present:

```text
observe → record → bridge/route → enforce → hand off verified history
```

Stronghold Net-Hunter owns the past:

```text
receive → verify → preserve → process → correlate → hunt → query → export
```

The system must preserve the distinction between authoritative packet history, structured records derived from that history, forwarding/enforcement disposition, and user-generated views/exports of that history.
