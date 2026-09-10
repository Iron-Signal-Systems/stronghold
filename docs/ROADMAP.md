# Stronghold Roadmap

## Roadmap Principle

Stronghold is one tightly coupled security platform composed of:

```text
Stronghold FW
Stronghold Net-Hunter
Stronghold Access
Stronghold Agent
```

Stronghold Access is the third infrastructure component when deployed and runs on a supported customer VM or bare-metal server. Stronghold Agent is the endpoint-side component managed through Access.

Iron Signal Systems Pathfinder is a separate ISS threat-intelligence system that may integrate as a first-class enrichment peer. It is not a Stronghold component and is not a Phase 0 dependency.

Implementation boundaries remain explicit even though the platform is tightly coupled.

> **Tightly coupled platform does not mean monolithic software.**

The immediate engineering priority remains proving the physical observation pipeline before expanding implementation into later architecture.

> **No more major product subsystems should be pulled into implementation until Phase 0 has measured the physical observation pipeline.**

# Phase 0 — Traffic Observation Foundation

Phase 0 proves that Stronghold FW can continuously capture configured physical interfaces, durably preserve traffic, truthfully report loss/degradation, catalog what was observed, manage local capture storage, and hand finalized history to Net-Hunter without allowing secondary work to compromise live capture.

**Routing, firewall enforcement, HA clustering, complete journal-integrity implementation, recovery/DR, encryption/key management, production platform-trust enforcement, complete appliance-update orchestration, complete Net-Hunter records/index/search/reprocessing, complete management-plane implementation, complete observability/alerting integrations, support/remote-engineering systems, Stronghold Access implementation, Agent endpoint enforcement, secure-access/VPN implementation, Pathfinder integration, and IDS/IPS are not part of Phase 0.**

Phase 0 is grounded in the Stronghold truth model:

```text
WHAT WAS PRESENTED
    authoritative capture

WHAT STRONGHOLD DID
    later authoritative operational / decision journals

WHAT STRONGHOLD UNDERSTOOD
    later derived interpretation / intelligence / enrichment
```

Phase 0 proves the first layer without requiring interpretation to become a capture prerequisite.

## Capture Requirement — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the Phase 0 reference visibility baseline.

Protocol decoding is not a prerequisite for capture. `not_decoded` must never be silently converted into `not_observed`.

## 0.1 Capture Segment Contract

Freeze:

```text
segment identity
source Appliance/interface provenance
open / finalized state
rotation triggers
start/end timestamps
packet/byte/drop accounting
PCAPNG requirements
storage state/location
integrity metadata
crash/incomplete semantics
durability boundary
catalog state before/after finalization
future clock-confidence compatibility
future HA physical-node provenance compatibility
future reprocessing without rewriting authoritative segments
```

Finalized/durable segments become eligible for transfer. Active segments remain safely inspectable only where doing so cannot jeopardize the writer.

The contract must account for the normal FW working-history model and outage backlog without turning Phase 0 into a complete retention system.

## 0.2 Segment and Traffic Catalog Contracts

Define the segment catalog used to locate packet data and the initial traffic/flow catalog used to answer what traffic occurred.

Avoid per-packet database records unless measurement later justifies them.

Unknown/undecoded packets remain authoritative capture content. The catalog must not make a parser-derived record the only proof that an authoritative segment exists.

Phase 0 catalog work must remain compatible with `docs/NET-HUNTER-RECORDS.md` without implementing the complete search/reprocessing system early.

## 0.3 PCAPNG Metadata Contract

Define metadata that belongs inside PCAPNG versus Stronghold catalogs, including:

```text
source appliance/interface provenance
timestamp resolution
segment identity
VLAN/offload representation
capture mode / relevant AF_XDP provenance
hardware/interface context needed to explain observation
loss/gap references where appropriate
```

## 0.4 FW Local Storage Contract

Freeze:

```text
RAM / AF_XDP UMEM working buffers
    → NVMe / XFS HOT
    → SSD / XFS WARM/BACKLOG where qualified
    → verified transfer to Net-Hunter
```

RAM/UMEM is never durable history. OS and PCAP filesystems remain separate.

The local storage model must support:

```text
NORMAL OPERATOR WORKING SET
    approximately 30–60 minutes

HUNTER OUTAGE CONDITION
    approximately 4 hours without Hunter = CRITICAL

QUALIFIED CAPACITY DIRECTION
    roughly 8 hours equivalent aggregate ingest workload
```

The 4-hour point is an operational critical threshold, not permission to delete unacknowledged history.

Capacity is based on aggregate bytes actually presented across configured capture interfaces, not a simplistic nominal firewall line-rate label.

Define `NORMAL`, `HIGH`, `URGENT`, `CRITICAL` pressure behavior, Hunter outage/backlog accounting, oldest-pending state, safe tier movement, and truthful exhaustion behavior.

Pressure may throttle secondary work and accelerate safe movement/transfer. It must not invent destructive authority.

## 0.5 Capture Resource-Protection Contract

Freeze Phase 0 workload priority:

```text
1. packet receive / AF_XDP queue service
2. active PCAP writes
3. segment finalization / essential integrity
4. essential observation/catalog/source-history state
5. local tier movement/backlog
6. transfer to Net-Hunter
7. compression where approved
8. deep indexing/analytics/enrichment outside FW live path
```

Define AF_XDP RX/FILL/COMPLETION ring occupancy, UMEM pressure, packet drops, CPU, memory, NVMe write latency/queue pressure, storage pressure, and transfer-backlog measurements used to throttle secondary work.

Future HA, journal checkpoint signing, updates, encryption overhead, support/diagnostics, observability, Access, Agent, secure-access, IDS/proxy, Pathfinder enrichment, and other later work must obey capture-first priority.

## 0.6 Single-Interface AF_XDP Capture Prototype

`docs/AF-XDP.md` defines the governing capture architecture.

```text
Arch Linux x86_64
    → one qualified physical NIC
    → minimal XDP program
    → AF_XDP socket bound to qualified RX queue
    → UMEM + RX/FILL/COMPLETION rings
    → PCAPNG writer
    → NVMe / XFS HOT
    → rotation
    → close / durability
    → hash
    → catalog
```

Capture without protocol allowlisting.

AF_XDP is the intended Phase 0 acquisition foundation. AF_PACKET/TPACKET_V3 is not the planned primary capture implementation and must not be used as a silent fallback while preserving an AF_XDP qualification claim.

Keep the XDP program minimal and measurable. Phase 0 does not turn XDP/eBPF into a second firewall or policy engine.

## 0.7 Multi-Interface / Multi-Queue AF_XDP Capture

Extend to multiple qualified physical interfaces and RX queues.

Begin with explicit queue ownership and a simple measurable socket/worker model.

Measure:

```text
queue-to-core affinity
RSS distribution
UMEM ownership/sharing
CPU locality
PCIe locality
NUMA placement
multi-interface contention
```

Do not freeze supported interface/queue count as a product claim until qualification proves it.

## 0.8 Performance, Loss, Visibility, and Qualification Baseline

Measure at minimum:

```text
Mbps/Gbps
packets/sec
CPU/core utilization
AF_XDP ring occupancy/pressure
UMEM pressure/starvation
copy/native/zero-copy mode
RSS/queue distribution
CPU affinity
NUMA/PCIe locality
memory
NVMe throughput/latency
NIC/XDP/AF_XDP/user-space/writer drops
segment finalization time
hashing time
catalog lag
sustained behavior
```

Exercise representative packet sizes and mixed profiles, including the small-packet/high-PPS case. The 64-byte packet case is a first-class stress profile.

Compare Stronghold visibility against Wireshark/dumpcap under equivalent supported interface conditions, including representative Layer-2/control-plane, VLAN, unknown, and malformed traffic when test infrastructure permits.

Preserve reproducible qualification context:

```text
Stronghold release/build
kernel
NIC/driver/firmware
XDP/AF_XDP mode
queue/RSS configuration
UMEM/ring sizes
CPU affinity
NUMA/PCIe topology
storage
traffic generator/profile
packet sizes
rate/PPS
duration
enabled features
capture config
loss counters/results
```

## 0.9 Local Tier Movement and Compression

Move finalized segments safely from HOT to WARM/BACKLOG where a separate tier is actually justified by measurement.

Evaluate Zstandard only where justified. Compression is secondary and must throttle/pause before capture suffers.

Verify lower-tier copy before removing higher-tier source.

Do not build a complex storage manager before Phase 0 proves that the HOT/WARM split provides operational value.

## 0.10 Verified Net-Hunter History Transfer

```text
FW finalized segment
    → integrity metadata
    → dedicated HISTORY transport
    → Hunter receive/finalize
    → Hunter independent verification
    → Hunter durable commit
    → Hunter ACK verified receipt
    → FW records acknowledgement
```

Production direction uses mTLS appliance identity plus explicit FW↔Hunter authorization and no plaintext fallback.

Hunter unavailable/full creates explicit backlog/degraded state. Hunter never ACKs uncommitted history.

Backlog recovery never outranks live packet acquisition or active PCAP writes.

# Phase 0 Exit Gate

Phase 0 is not complete until representative supported hardware can continuously capture configured interfaces for an extended period and Stronghold can truthfully answer:

> **What did the hardware present to Stronghold, what did Stronghold durably capture, what—if anything—was dropped, where are the packets now, what history is pending transfer, what has Net-Hunter independently verified/committed, and can the relevant traffic be found/exported without compromising ongoing capture?**

The gate includes:

```text
Wireshark/dumpcap visibility comparison
AF_XDP queue/UMEM behavior
copy/native/zero-copy truthfulness
L2/control-plane visibility
load/loss accounting
64-byte/small-packet PPS stress
CPU/NUMA/PCIe locality
interruption recovery
storage pressure
tier movement
Hunter outage/backlog
destination-full behavior
verified handoff
```

# Architecture Defined for Later Phases

The following architecture is already established conceptually but is **not** implementation authorization for Phase 0.

## Stronghold FW / Networking

```text
Layer-2 transparent bridging
Layer-3 IPv4/IPv6 routing
router-on-a-stick / hybrid networking
default-deny policy
early authorization before normal forwarding work
explicit NOT_PERFORMED states
NAT
VLAN / zone / interface object model
FQDN truth separation
policy-defined multi-WAN preference
stateful enforcement/reconciliation
policy simulation / counterfactual testing
transactional configuration generations
```

## Journals / History / External Logging

```text
separate append-oriented journal domains
canonical exact-byte journal integrity representation
durable journal commit/batching
periodic signed checkpoints/finalized segments
journal epochs and continuity-gap handling
Hunter verification / external anchoring direction
independent retention / holds / destruction
structured journal export to SIEM/log systems
Security Onion / Graylog integration direction
SIEM receipt separated from Hunter durable-history ACK
```

## Net-Hunter

```text
FreeBSD / ZFS / jails
verified authoritative PCAP/journal history
segment catalog vs traffic/record catalog
no row-per-packet primary search requirement
historical relationship preservation
explicit direct/correlated/external provenance
query-coverage state
index fallback toward segment catalogs/PCAP
versioned targeted/full historical reprocessing
configuration-to-decision correlation
traffic-derived result pivot back toward PCAP
future IDS / Inspection Jail
Pathfinder retrospective enrichment/reprocessing
```

## Stronghold Access

`docs/access/ARCHITECTURE.md` is authoritative for component ownership.

Stronghold Access is the third Stronghold infrastructure node and is intended to run on a supported customer VM or bare-metal server.

Architecture includes:

```text
identity inputs
device trust / posture
first-class 802.1X / EAP direction
AAA integration
Network Access Sessions
Stronghold Access Sessions
resource authorization
MFA
revocation / reevaluation
signed Agent policy generations
network context from FW/admission
controlled FW integration
Pathfinder intelligence/risk input
```

## Stronghold Agent

`docs/agent/ARCHITECTURE.md` is authoritative for endpoint component ownership.

Architecture includes:

```text
Windows-first Go service
Windows-native APIs
Windows Filtering Platform endpoint PEP
process/application-aware connection enforcement
endpoint identity / posture
signed policy generations
local-network and remote/ZTNA operation
Protected Endpoint mode
WireGuard transport participation
endpoint decision records
Hunter correlation
no direct Pathfinder dependency by default
```

Deep payload-aware endpoint L7 remains a separate later qualification decision.

## Secure Access

Remote access:

```text
NIST SP 800-207 PE / PA / PEP direction
Stronghold Access as control authority
Stronghold Agent endpoint PEP
Stronghold FW network PEP
WireGuard secure transport/data plane
resource-scoped authorization
first-class GRANT / DENY / REVOKE
```

Site-to-site / branch office:

```text
L2TPv3
IPsec protection
site/tunnel identity
VLAN / bridge / routing integration
normal Stronghold policy enforcement
```

Secure-access implementation remains deferred.

## IDS / TLS Inspection

```text
selective TLS/application proxying
FW encrypted-wire PCAP remains authoritative
FW plaintext volatile-only
purpose-specific encrypted INSPECTION transport
isolated Hunter IDS / Inspection Jail
encrypted-at-rest findings/context
findings-only default direction
explicit inspection coverage/gaps
Pathfinder intelligence complements IDS/IPS
IDS + Pathfinder context never automatically creates IPS authority
```

IDS/IPS implementation remains deferred.

## Pathfinder Integration

`docs/PATHFINDER-INTEGRATION.md` governs the relationship.

Pathfinder is a separate ISS system, not a Stronghold component.

Stronghold may use Pathfinder for:

```text
FW observable / operator enrichment
future explicit policy inputs
IDS/IPS contextual enrichment
Net-Hunter retrospective matching/reprocessing
Stronghold Access risk/trust reevaluation
qualified Stronghold-to-Pathfinder observable submission
```

Mandatory boundaries include:

```text
Pathfinder match                    != Stronghold enforcement action
Pathfinder malicious classification != compromise proven
Pathfinder risk signal              != automatic Access REVOKE
IDS finding                         != Pathfinder confirmation
Stronghold observation              != Pathfinder intelligence
retrospective match                 != historical real-time detection
```

Pathfinder integration must not become an unbounded synchronous per-packet FW dependency.

## Management / Governance

```text
single Stronghold management authority
CLI/API/future UI parity
candidate/validate/diff/simulate/commit
mandatory change reason
Git-backed configuration lineage
stable object identity
stale-candidate/concurrency protection
configuration vs operational vs historical state
native platform drift detection
least privilege / scoped authority
```

## HA

```text
active/standby FW
Cluster ID + cluster-owned forwarding identity
node-local physical capture provenance
cluster vs node-local config
session/NAT/WAN state synchronization
heartbeat/control separated from bulk sync
conservative promotion/fencing/split-brain prevention
secure-access/tunnel continuity separately qualified
```

## Appliance Lifecycle / Recovery / Crypto / Platform Trust

```text
Stronghold-controlled Arch appliance lifecycle
signed online/offline update bundles
standby-first rolling upgrades
controlled schema migration / rollback
layered FreeBSD/Hunter update model
ZFS irreversible-feature boundary
recovery / replacement / DR state model
configuration vs identity/secret recovery
offline failed-node PCAP recovery
purpose-separated at-rest encryption domains
TPM/HSM normal protection with separate recovery authority
non-exportable private keys where supported
key rotation / compromise / crypto-shred authority
Secure Boot / measured boot / attestation separation
qualified appliance profiles
NIC/driver/firmware and PCIe/NUMA qualification
```

# Later Qualification Gates

## Stronghold Access Qualification

Before production Access implementation, freeze and validate:

```text
supported host OS / VM / bare-metal profiles
identity provider model
AAA/802.1X/EAP interoperability
Network Access Session schema
Stronghold Access Session schema
MFA / posture
policy language / generations
Agent control protocol
FW control protocol
Pathfinder risk/intelligence input contract
mTLS / trust lifecycle
replay/downgrade protection
lease/holdover/expiration
revocation / CoA semantics
HA / recovery / backup
journaling / observability
scale / latency / load
upgrade / rollback
```

## Agent Qualification

Before production Agent implementation, freeze and validate:

```text
supported Windows versions
Go service lifecycle / privileges
installer/update/signing
endpoint enrollment / identity
policy bundle schema/signing
lease/holdover/expiration
WFP integration/layers/filter ownership
application/process identity semantics
user/device identity semantics
network-context authority
Access Session integration
WireGuard integration
Agent/FW dual-PEP semantics
endpoint decision records
Pathfinder-influenced policy provenance
local-admin tamper boundary
performance / latency
sleep/resume / multi-NIC / network transition
failure/recovery
Hunter correlation
```

## Pathfinder Qualification

Before production Stronghold↔Pathfinder integration, freeze and validate:

```text
trust/enrollment
API/schema/versioning
observable normalization
Pathfinder Record/reference semantics
classification/confidence/freshness
provenance/lineage
FW caching/update model
synchronous lookup prohibition/boundary
Access risk-input semantics
IDS correlation semantics
Hunter retrospective matching/reprocessing
Stronghold-to-Pathfinder submission policy
privacy/data governance
backpressure/rate limiting
failure/holdover/stale-data behavior
journaling
credential purpose/rotation/revocation
performance/scaling
upgrade/rollback compatibility
```

## IDS / TLS Inspection Qualification

Before implementation, freeze and validate the selective-inspection policy model, proxy behavior, customer inspection CA, origin certificate validation, pinned/mTLS/QUIC behavior, volatile-only FW plaintext handling, encrypted inspection transport, Hunter jail isolation, findings schema/retention, Pathfinder correlation, inspection coverage, overload/failure behavior, and any explicit IPS fail-closed policy.

## HA Qualification

Validate forwarding failover, configuration synchronization, session continuity, split-brain prevention/fencing, cluster virtual identity ownership, capture continuity/gaps, failed-node history reconciliation, journal truthfulness, platform-trust eligibility, and later secure-access/tunnel state separately.

## Update / Recovery / Crypto / Hardware Qualification

Freeze and validate release signing, compatibility windows, schema migration, rollback, recovery keys, hardware replacement, key recovery, platform trust, NIC/AF_XDP matrices, NUMA/PCIe topology, sustained NVMe behavior, feature-specific performance, and HA-pair performance.

# Remaining Architecture Requiring Deliberate Freezing

Major areas still not fully frozen include:

```text
exact cryptographic algorithm/module profile
exact recovery-key authority/mechanics
exact journal canonical encoding/signature algorithms
exact external journal witness/anchor design if adopted
exact HA fencing/election implementation
exact supported hardware profiles
Net-Hunter off-system backup/replication/archive architecture
exact Net-Hunter database/index technology and physical schemas
exact Net-Hunter query language/API/partitioning strategy
installation / factory provisioning / first-boot bootstrap architecture
later dynamic-routing implementation contracts
exact future Layer-7 boundaries beyond defined principles
Stronghold Access implementation
Stronghold Agent implementation
secure-access implementation
Pathfinder integration implementation
IDS/IPS implementation
```

The exact later implementation sequence remains intentionally unfrozen until Phase 0 produces real capture/storage/performance data.

No later capability is pulled into Phase 0 without explicit approval.
