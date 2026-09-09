# Stronghold AF_XDP Capture Architecture

## Purpose

This document records the packet-acquisition decision for Stronghold FW Phase 0 and later qualified dataplane work.

> **AF_XDP is the intended Stronghold FW packet-acquisition foundation.**

AF_XDP is not treated as an optional optimization to be considered after an AF_PACKET implementation. Phase 0 capture engineering begins with AF_XDP and qualifies the exact NIC/driver/kernel/queue behavior required to preserve Stronghold's physical-interface observation invariant.

## Governing Capture Invariant

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

AF_XDP is an implementation path beneath this product invariant. It does not weaken or redefine the visibility requirement.

```text
AF_XDP selected
!=
all observable traffic proven captured
```

Stronghold must still account truthfully for any visibility change caused by NIC hardware, driver behavior, XDP attachment mode, queue steering, RSS, VLAN/checksum offload, timestamp behavior, ring exhaustion, user-space starvation, or other platform behavior.

## Initial Capture Path

Conceptual Phase 0 path:

```text
qualified physical NIC
        ↓
RX queues / RSS
        ↓
minimal XDP attachment
        ↓
AF_XDP sockets
        ↓
UMEM
        ↓
Stronghold capture workers
        ↓
PCAPNG writer
        ↓
NVMe / XFS HOT
```

Phase 0 should keep the XDP program as small and measurable as practical. XDP/eBPF must not silently become a second firewall-policy engine before the authoritative capture path is proven.

## AF_XDP Modes

Stronghold distinguishes AF_XDP availability from qualified operating mode.

Conceptually:

```text
XDP_SKB / generic-copy path
XDP_DRV / native-driver path
AF_XDP zero-copy where NIC/driver support is qualified
```

Exact accepted modes for each production appliance profile are qualification decisions.

Development may use a copy-capable AF_XDP path where necessary, but a production performance claim must identify the actual mode tested.

```text
AF_XDP available
!=
AF_XDP zero-copy available

AF_XDP zero-copy available
!=
Stronghold zero-copy qualified
```

Stronghold must never silently fall back to a materially different capture mode while continuing to present the old performance/qualification state as though nothing changed.

## Queue Ownership and Scaling

AF_XDP is queue-oriented. Stronghold therefore treats NIC queue topology as a first-class qualification concern.

Phase 0 and later performance work must measure, as applicable:

```text
RX queue count
RSS behavior
queue-to-socket binding
queue-to-worker ownership
queue-to-core affinity
UMEM ownership/sharing
fill/completion ring behavior
RX/TX ring sizing where applicable
batch size
wake-up behavior
queue starvation
queue-specific drops
```

Multi-interface and multi-queue scaling must be measurement-driven. The implementation should begin with explicit ownership rather than hiding queue behavior behind a generalized worker framework.

## CPU, NUMA, and PCIe Locality

On multi-socket systems, Stronghold must account for locality among:

```text
NIC PCIe attachment
NUMA node
RX queue
AF_XDP socket
UMEM allocation
capture worker CPU
storage/HBA/NVMe path where relevant
```

Avoid unnecessary cross-NUMA traffic where the qualified hardware topology permits local placement.

```text
NIC on NUMA 0
+
UMEM on NUMA 1
+
worker on NUMA 1
!=
qualified optimal locality
```

Single-socket development systems remain useful for correctness and initial bring-up; dual-socket systems are valuable for discovering queue/locality/PCIe behavior that simpler hardware will not expose.

## Visibility and Offload Qualification

AF_XDP qualification must explicitly test visibility-affecting behavior, including where applicable:

```text
VLAN tag representation
checksum offload representation
RSS hashing / queue distribution
promiscuous/filter behavior
MTU and jumbo frames
unknown EtherTypes
malformed traffic
Layer-2/control-plane traffic
hardware/software timestamps
NIC reset/recovery behavior
XDP attach/detach behavior
```

Wireshark/dumpcap remains the reference visibility comparison under equivalent supported physical-interface conditions.

A faster AF_XDP configuration is not accepted if it produces unexplained observation differences.

## Drop and Gap Accounting

AF_XDP does not remove the requirement for end-to-end loss accounting.

Stronghold must distinguish measurable loss/failure points such as:

```text
NIC / hardware drop
XDP path drop
AF_XDP RX-ring starvation
fill-ring starvation
UMEM/resource exhaustion
capture-worker backlog/drop
PCAP writer/storage failure
intentional policy drop later in forwarding
```

Exact counter availability depends on qualified NIC/driver/kernel behavior. Unavailable measurement is reported as unavailable/unknown rather than guessed.

```text
packet absent from PCAP
!=
policy dropped packet
```

unless Stronghold can actually establish that causal fact.

## Resource Priority

AF_XDP packet acquisition remains the highest live-data priority on FW.

Conceptual ordering remains:

```text
1. AF_XDP packet acquisition / ring service
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation/decision/journal state
5. critical HA heartbeat/control where later implemented
6. journal checkpoint/finalization
7. local tier movement/backlog
8. HA bulk state sync
9. authoritative Hunter HISTORY transfer
10. compression
11. IDS/analytics/observability/support work
```

Secondary work must throttle before AF_XDP ring starvation or packet loss is hidden.

## Phase 0 Performance Work

Phase 0 must qualify packet rate as seriously as bandwidth.

Testing includes, as applicable:

```text
64-byte packet stress
mixed packet sizes
large frames
bidirectional traffic
multiple RX queues
multiple interfaces
sustained duration
high flow count
new-flow bursts
storage contention
Hunter-transfer contention
CPU/NUMA locality
```

A clean 10-Gb/s large-frame throughput result is not sufficient proof of a 10-Gb/s Stronghold capture profile.

## Hardware Qualification

AF_XDP capability belongs to the qualified appliance profile and is not inferred only from kernel support.

Qualification binds at least:

```text
Stronghold release
Linux kernel
NIC model
NIC firmware
NIC driver
XDP/AF_XDP operating mode
queue configuration
RSS configuration
CPU topology
NUMA/PCIe topology
UMEM/ring configuration
capture-worker placement
storage profile
```

```text
NIC supports AF_XDP
!=
NIC qualified for Stronghold
```

A driver or firmware update that changes AF_XDP behavior requires appropriate regression qualification before becoming part of a supported Stronghold release/profile.

## Failure Behavior

If the configured/qualified AF_XDP path cannot be established, Stronghold must report the actual state.

The production appliance must not silently substitute AF_PACKET/TPACKET_V3 or another packet-acquisition mechanism while claiming the AF_XDP-qualified profile remains active.

Any future alternate acquisition path requires explicit architectural/qualification approval and must preserve the same observation and failure-state requirements.

## Implementation Discipline

Phase 0 should prefer a small, explicit AF_XDP native boundary over premature abstraction.

Native code must validate AF_XDP/XDP/kernel-returned metadata, descriptor lengths, offsets, ring indices, buffer ownership, queue identity, and termination/error conditions before converting them into Stronghold facts.

Go implementation follows `go/AGENTS.md`. Linux-native bindings should prefer direct supported interfaces such as `golang.org/x/sys/unix` where practical. Any additional AF_XDP/eBPF dependency must be justified, reviewable, and consistent with ISS dependency/security standards.

## Truth Separations

```text
AF_XDP selected                 != capture proven correct
AF_XDP available                != zero-copy available
zero-copy available             != zero-copy qualified
higher throughput               != better observation
link speed                      != qualified capture rate
Gbps                            != PPS capability
queue configured                != queue serviced adequately
worker running                  != ring healthy
ring healthy now                != no historical capture gap
NIC supported by vendor         != Stronghold-qualified NIC
```

## Scope

This document changes the Phase 0 packet-acquisition foundation to AF_XDP. It does not pull routing, firewall enforcement, HA, IDS/IPS, TLS proxying, or other later product systems into Phase 0.
