# Stronghold Roadmap

## Phase 0 — Traffic Observation Foundation

Phase 0 exists to prove that Stronghold can continuously capture configured interfaces, durably preserve the traffic, truthfully report loss and degradation, catalog what was observed, and manage capture storage without allowing secondary work to compromise live capture.

Routing and firewall enforcement are not part of Phase 0.

### 0.1 Capture segment contract

Freeze the exact capture-segment record and lifecycle.

Define:

- segment identity;
- interface identity/context;
- start/end timestamps;
- packet/byte/drop accounting;
- PCAPNG format requirements;
- storage state/location;
- integrity metadata;
- allowed lifecycle transitions; and
- what constitutes a successfully durable and verified segment.

### 0.2 Segment and traffic catalog contracts

Define the segment catalog used to locate packet data.

Define the initial traffic/flow catalog used to answer what traffic occurred.

Avoid per-packet database records unless a later measured requirement justifies them.

### 0.3 PCAPNG metadata contract

Define which metadata must travel inside the PCAPNG itself versus the Stronghold catalog.

A capture copied away from the appliance should retain useful provenance such as interface/capture context, timestamp resolution, and Stronghold segment identity where the format safely permits it.

### 0.4 Storage tier contract

Freeze the downward storage model:

```text
RAM buffer
    -> NVMe HOT
    -> SSD WARM
    -> HDD/RAID COLD
    -> optional ARCHIVE/OFFLOAD
```

Define high-water, urgent, and critical storage-pressure behavior.

RAM is never classified as durable capture storage.

### 0.5 Capture resource-protection contract

Freeze the workload priority:

```text
1. packet receive
2. active PCAP writes
3. segment finalization / essential integrity work
4. essential catalog state
5. tier migration
6. compression
7. remote offload
8. deep indexing / analytics
```

Define the measurements that throttle or pause secondary work, including capture-ring occupancy, packet drops, CPU load, memory pressure, NVMe write latency/queue pressure, storage pressure, and backlog.

### 0.6 Single-interface capture prototype

Implement the smallest useful capture path:

```text
Arch Linux x86_64
    -> one physical NIC
    -> AF_PACKET / TPACKET_V3
    -> RAM ring
    -> PCAPNG writer
    -> NVMe hot storage
    -> rotation
    -> close / durability boundary
    -> hash
    -> catalog
```

No compression, tiering, offload, routing, or firewall code is required for this slice.

### 0.7 Multi-interface capture

Extend capture to the initial target of 2–4 1 GbE interfaces.

Begin with one capture worker/ring per physical interface unless profiling establishes a need for a different arrangement.

### 0.8 Performance and loss profiling

Test representative sustained rates and packet sizes.

Measure at minimum:

- Mbps/Gbps;
- packets per second;
- CPU;
- memory/ring occupancy;
- NVMe throughput and latency;
- capture drops;
- segment-finalization time;
- hashing time; and
- catalog lag.

Exercise traffic loads such as 100, 250, 500, 750, and 1000 Mb/s where practical, with small packets, standard 1500-byte packets, and mixed traffic profiles.

Optimization such as PACKET_FANOUT, queue-aware scaling, CPU affinity, AF_XDP, or other techniques should be justified by measurements rather than added preemptively.

### 0.9 Compression and downward tiering

Add background downward tiering of finalized segments.

Evaluate Zstandard as the initial compression method.

Compression must occur only on finalized segments and must automatically throttle/pause when capture resources require priority.

Verify the lower-tier copy before removing the higher-tier source.

### 0.10 Selective VLAN offload

Add downstream selection/offload of configured VLAN traffic from authoritative local capture.

Initial destination classes to design/test:

- SMB / UNC-backed storage;
- SFTP;
- SSH-based transfer; and
- iSCSI-backed mounted storage.

Offload destination failure must create an observable backlog/degraded condition without stopping local capture.

## Phase 0 Exit Gate

Phase 0 is not complete until Stronghold can demonstrate, on representative supported hardware, that it can continuously capture configured interfaces for an extended period and truthfully answer:

> **What did the hardware present to Stronghold, what did Stronghold durably capture, what—if anything—was dropped, where are the packets now, and can the relevant traffic be found and exported without compromising ongoing capture?**

The exit gate must include documented resource/load behavior, loss accounting, interrupted-operation recovery, storage-pressure behavior, migration verification, and offload-degradation behavior.

## Later Phases

Later roadmap phases will cover networking and enforcement, including static routing, VLAN/subinterface configuration, nftables stateful policy, policy objects, and FQDN-resolved policy objects.

Those phases must not be pulled into Phase 0 without explicit approval.
