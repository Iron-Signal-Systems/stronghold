# Stronghold Architecture

## Purpose

This document records the current high-level architecture for Stronghold's capture-first foundation. It intentionally describes only the architecture agreed for the present phase and the immediate layers that depend on it.

## Governing Principle

> **Capture first. Never sacrifice observation for secondary work.**

Stronghold must prioritize receiving packets and durably writing active capture data over compression, tier migration, offload, deep indexing, and analytics.

## Capture Invariant #1 — Wireshark-Class Interface Visibility

Stronghold's first capture requirement is:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, routes, or firewall-processes it.**

The authoritative observation point is the configured supported physical interface, not a higher-level assumption about what Linux routes or what a firewall rule understands.

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

NIC offloads and driver behavior that can alter the userspace representation of traffic, including VLAN metadata handling and packet aggregation, must be explicitly evaluated during hardware qualification and Phase 0 testing.

## Initial Appliance

```text
Platform:        Arch Linux
Architecture:    x86_64 / amd64
Interfaces:      2–4 × 1 GbE
Administration:  CLI first
Capture:         Continuous full-packet capture
Initial source:  AF_PACKET / TPACKET_V3
Format:          PCAPNG
```

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
NVMe hot ingest tier
 │
 ▼
closed capture segment
```

RAM is a transient buffering tier only. A packet is not considered durably captured merely because it reached RAM.

The authoritative configured capture stream is local-first. Downstream filtering for retention or offload must not redefine what Stronghold originally captured.

## Segment Lifecycle

The exact implementation contract remains to be frozen, but the intended lifecycle is:

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
TIER_ELIGIBLE
```

Only finalized closed segments are eligible for downward tiering.

## Storage Tiers

Preferred storage moves downward in performance class:

```text
RAM        buffer only
  ↓
NVMe       HOT / ingest
  ↓
SSD        WARM
  ↓
HDD/RAID   COLD
  ↓
ARCHIVE    optional remote/offload destination
```

Traffic load changes residence time and drain behavior, not the primary ingest path.

### Migration rule

For a downward move:

```text
read source segment
    ↓
write destination
    ↓
finalize destination
    ↓
verify destination
    ↓
update catalog location/state
    ↓
remove source only when safe
```

A source segment must never be deleted solely because a copy call returned successfully.

## Compression

Compression occurs during downward tiering, not while packets are actively being captured.

Zstandard is the current preferred direction for evaluation. The exact level and policy will be determined by measurement.

Compression is opportunistic and subordinate to capture. When CPU, capture-ring occupancy, packet drops, free RAM, NVMe write latency, or other capture-critical indicators show pressure, compression must throttle or pause.

The preferred behavior is to compress once at the earliest safe downward transition and then move the already-compressed object through lower tiers without recompressing it.

## Catalogs

Stronghold is expected to maintain two related views.

### Segment catalog

Answers: **Where are the packets?**

Expected fields include:

```text
segment_id
capture_host
interface
start_time
end_time
packet_count
byte_count
drop_count
format
storage_tier
storage_location
compression
hash
status
```

### Traffic / flow catalog

Answers: **What traffic happened?**

Expected initial flow metadata includes:

```text
first_seen
last_seen
source_ip
source_port
destination_ip
destination_port
protocol
interface
vlan
packets
bytes
segment_id
```

Stronghold should not create one database row per packet merely to make traffic searchable. Flow/session metadata points back to the authoritative capture segment containing the packets.

Unknown or undecoded traffic remains in the authoritative capture even when the traffic catalog cannot yet provide protocol-specific enrichment for it.

## Offload

Capture policy and offload policy are separate.

Example:

```text
capture interfaces:
    eth0
    eth1

offload selection:
    VLAN 1
    VLAN 2
    VLAN 201
```

Stronghold captures the configured local stream first, then derives/selects traffic for configured long-term destinations.

Planned destination classes:

```text
SMB / UNC-backed storage
SFTP
SSH-based transfer
iSCSI-backed mounted storage
```

Remote destination failure must not stop local capture. It creates offload backlog and an observable degraded state.

Derived/offloaded captures must preserve provenance back to source segment(s) and selection criteria.

## Resource Priority

Stronghold's intended scheduling and resource-priority model is:

```text
1. packet acquisition
2. active PCAP writes to hot storage
3. segment finalization and minimum integrity work
4. essential catalog state
5. tier migration
6. compression
7. remote offload
8. deep indexing / analytics
```

Secondary work is eventually consistent. Capture is real-time.

If Stronghold observes capture loss or dangerous ring/storage pressure, nonessential work must be capable of throttling or pausing automatically.

## Future Layers

The following are intentionally later than the capture foundation:

- static routing;
- VLAN/subinterface configuration;
- nftables stateful firewall policy;
- source/destination/service objects;
- FQDN-resolved policy objects;
- firewall-rule correlation with captured traffic; and
- additional security features only when explicitly added to the roadmap.

Stronghold should not implement these early merely because the capture architecture can eventually support them.
