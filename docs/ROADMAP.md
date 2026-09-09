# Stronghold Roadmap

## Phase 0 — Traffic Observation Foundation

Phase 0 proves that Stronghold can continuously capture configured physical interfaces, durably preserve traffic, truthfully report loss/degradation, catalog what was observed, manage local capture storage, and hand finalized history to Net-Hunter without allowing secondary work to compromise live capture.

**Routing, firewall enforcement, HA clustering, complete journal-integrity implementation, recovery/DR, encryption/key management, production platform-trust enforcement, complete appliance-update orchestration, complete Net-Hunter records/index/search/reprocessing, complete management-plane implementation, complete observability/alerting integrations, support/remote-engineering systems, secure-access/VPN implementation, and IDS/IPS are not part of Phase 0.**

Phase 0 is grounded in the Stronghold truth model:

```text
WHAT WAS PRESENTED
    authoritative capture

WHAT STRONGHOLD DID
    later authoritative operational / decision journals

WHAT STRONGHOLD UNDERSTOOD
    later derived interpretation
```

Phase 0 proves the first layer without requiring interpretation to become a capture prerequisite.

### Capture Requirement #1 — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the Phase 0 reference visibility baseline.

Protocol decoding is not a prerequisite for capture. `not_decoded` must never be silently converted into `not_observed`.

### 0.1 Capture Segment Contract

Freeze:

- segment identity;
- source Appliance/interface provenance;
- start/end timestamps;
- packet/byte/drop accounting;
- PCAPNG requirements;
- storage state/location;
- integrity metadata;
- allowed lifecycle transitions;
- durability/verification boundary;
- future clock-confidence compatibility;
- future HA physical-node provenance compatibility; and
- future reprocessing without rewriting authoritative segments.

### 0.2 Segment and Traffic Catalog Contracts

Define the segment catalog used to locate packet data and the initial traffic/flow catalog used to answer what traffic occurred.

Avoid per-packet database records unless measurement later justifies them.

Unknown/undecoded packets remain authoritative capture content. The catalog must not make a parser-derived record the only proof that an authoritative segment exists.

Do not force future Administrative, System/Health, Traffic Decision, Trust/Identity, Time/Clock, or Hunter Processing Journals into one mutable generic logging store.

Phase 0 catalog work must remain compatible with the later `docs/NET-HUNTER-RECORDS.md` authority model without implementing the complete search/reprocessing system early.

### 0.3 PCAPNG Metadata Contract

Define metadata that belongs inside PCAPNG versus Stronghold catalogs, including source/interface provenance, timestamp resolution, segment identity, and VLAN/offload metadata required to preserve what the supported NIC/driver actually presented.

### 0.4 FW Local Storage Contract

Freeze:

```text
RAM / AF_XDP UMEM working buffers
    → NVMe / XFS HOT
    → SSD / XFS WARM/BACKLOG
    → verified transfer to Net-Hunter
```

RAM/UMEM is never durable history. OS and PCAP filesystems remain separate.

Define `NORMAL`, `HIGH`, `URGENT`, `CRITICAL` pressure behavior, Hunter outage/backlog accounting, oldest-pending state, safe tier movement, and truthful exhaustion behavior.

Pressure may throttle secondary work and accelerate safe movement/transfer. It must not invent destructive authority or silently delete unacknowledged authoritative history merely to hide a full disk.

### 0.5 Capture Resource-Protection Contract

Freeze Phase 0 workload priority:

```text
1. packet receive / AF_XDP queue service
2. active PCAP writes
3. segment finalization / essential integrity
4. essential observation/catalog/source-history state
5. local tier movement/backlog
6. transfer to Net-Hunter
7. compression where approved
8. deep indexing/analytics outside FW live path
```

Define AF_XDP RX/fill/completion ring occupancy, UMEM pressure, packet drops, CPU, memory, NVMe write latency/queue pressure, storage pressure, and transfer-backlog measurements used to throttle secondary work.

Future HA heartbeat, journal checkpoint signing, update work, encryption overhead, support/diagnostic work, observability collection, IDS/proxy work, secure-access work, and other later systems must obey capture-first priority.

### 0.6 Single-Interface AF_XDP Capture Prototype

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

Keep the XDP program minimal and measurable. Phase 0 does not turn XDP/eBPF into a second firewall or policy engine before the observation path is proven.

No routing, firewall enforcement, HA, complete journal subsystem, retention administration, complete update manager, DR, key management, complete Hunter record/index/search/reprocessing system, complete management plane, complete observability/alerting system, support/remote-engineering system, secure-access/VPN implementation, IDS/IPS, TLS/application proxying, UI, or other later system is required here.

### 0.7 Multi-Interface / Multi-Queue AF_XDP Capture

Extend to multiple qualified physical interfaces and RX queues.

Begin with explicit queue ownership and a simple measurable socket/worker model. Queue-to-core affinity, RSS distribution, UMEM ownership/sharing, CPU locality, PCIe locality, and NUMA placement must be measured rather than hidden behind a generic worker pool.

Do not freeze supported interface/queue count as a product claim until qualification proves it.

### 0.8 Performance, Loss, Visibility, and Qualification Baseline

Measure at minimum:

- Mbps/Gbps;
- packets/sec;
- CPU/core utilization;
- AF_XDP RX/FILL/COMPLETION ring occupancy/pressure;
- UMEM pressure and starvation;
- AF_XDP copy/native/zero-copy operating mode;
- RSS/queue distribution;
- CPU affinity and NUMA/PCIe locality where applicable;
- memory;
- NVMe throughput/latency;
- NIC/XDP/AF_XDP/user-space/writer drops where measurable;
- segment finalization time;
- hashing time;
- catalog lag; and
- sustained behavior over representative duration.

Exercise representative packet sizes and mixed profiles, including small-packet/high-PPS cases. Queue topology, affinity, batch size, ring size, UMEM sizing, zero-copy use, and other tuning must be justified by measurement.

Compare Stronghold visibility against Wireshark/dumpcap under equivalent supported interface conditions, including representative Layer-2/control-plane, VLAN, unknown, and malformed traffic when test infrastructure permits.

Preserve enough reproducible test context to become a later Stronghold release/hardware qualification baseline:

```text
Stronghold release/build
kernel
NIC/driver/firmware
XDP/AF_XDP mode
queue/RSS configuration
UMEM/ring sizes
CPU affinity
NUMA/PCIe topology context
storage
traffic generator/profile
packet sizes
rate/PPS
duration
enabled features
capture config
loss counters/results
```

Phase 0 records qualification data; it does not implement the later appliance-profile/support system.

### 0.9 Local Tier Movement and Compression

Move finalized segments safely from HOT to WARM/BACKLOG as required.

Evaluate Zstandard where justified. Compression is secondary and must throttle/pause before capture suffers.

Verify lower-tier copy before removing higher-tier source.

### 0.10 Verified Net-Hunter History Transfer

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

Production direction uses mTLS appliance identity plus explicit FW↔Hunter authorization and no plaintext fallback. Exact Phase 0 trust/TLS slice must be frozen before implementation; do not invent an insecure temporary path.

Hunter unavailable/full creates explicit backlog/degraded state. Hunter never ACKs uncommitted history.

## Phase 0 Exit Gate

Phase 0 is not complete until representative supported hardware can continuously capture configured interfaces for an extended period and Stronghold can truthfully answer:

> **What did the hardware present to Stronghold, what did Stronghold durably capture, what—if anything—was dropped, where are the packets now, what history is pending transfer, what has Net-Hunter independently verified/committed, and can the relevant traffic be found/exported without compromising ongoing capture?**

The exit gate includes Wireshark/dumpcap visibility comparison, AF_XDP queue/UMEM behavior, copy/native/zero-copy mode truthfulness, L2/control-plane behavior, load/loss accounting, small-packet/PPS stress, CPU/NUMA/PCIe locality, interruption recovery, storage pressure, tier movement, Hunter outage/backlog, destination-full behavior, and verified handoff.

## Later Architecture Already Defined at Concept Level

The following concepts are architecturally established but not yet decomposed into implementation phases:

```text
Layer-2 transparent bridging
Layer-3 IPv4/IPv6 routing
router-on-a-stick / hybrid networking
default-deny security policy
early authorization before normal forwarding work
explicit NOT_PERFORMED states
NAT
VLAN / zone / interface object model
FQDN truth separation
policy-defined multi-WAN preference
transactional configuration generations
LDAPS-only AD authentication direction
RADIUS / TACACS+ direction
RBAC / MFA direction
stable Appliance ID
mTLS FW↔Hunter trust + explicit peer authorization
UTC + independent ordering / clock confidence
separate append-oriented journal domains
independent retention / holds / destruction
Net-Hunter four-jail architecture
active/standby FW HA
Cluster ID + cluster-owned forwarding identity
node-local physical capture provenance
cluster vs node-local config
session/NAT/WAN state synchronization
heartbeat/control separated from bulk sync
conservative promotion/fencing/split-brain prevention
Stronghold-controlled Arch appliance lifecycle
signed online/offline update bundles
standby-first rolling upgrades
controlled schema migration / rollback
layered FreeBSD/Hunter update model
ZFS irreversible-feature boundary
recovery / replacement / DR state model
separate configuration vs identity/secret recovery
offline failed-node PCAP recovery
Hunter recovery with destructive operations enabled last
purpose-separated at-rest encryption domains
TPM/HSM normal protection with separate recovery authority
ZFS-native Hunter encrypted-dataset direction
non-exportable appliance private keys where supported
key rotation / compromise / crypto-shred authority model
independent hash-linked journal streams
canonical exact-byte journal integrity representation
durable journal commit/batching
periodic signed journal checkpoints/finalized segments
journal epochs and continuity-gap handling
Hunter verification / external anchoring of FW journal heads
Secure Boot / measured boot / attestation separation
logical identity vs hardware/root-of-trust identity
future platform-trust state
qualified appliance profiles
AF_XDP as the intended FW acquisition foundation
AF_XDP queue/UMEM/RSS/affinity/NUMA/PCIe qualification
AF_XDP copy/native/zero-copy state separated from qualification
NIC/driver/firmware and PCIe/NUMA qualification
separate capture/full-feature/HA performance claims
Net-Hunter sustainable ingest/query/degraded-storage qualification
Net-Hunter segment catalog vs traffic/record catalog separation
derived records/indexes rebuildable from authoritative history
no row-per-packet primary search requirement
historical MAC/IP/name/VLAN relationship preservation
explicit direct-vs-correlated derived provenance
explicit search/query coverage state
index fallback toward segment catalogs/PCAP
versioned targeted/full historical reprocessing
processing/index backlog separation
historical configuration-to-decision correlation
traffic-derived result pivot back toward PCAP
single Stronghold management authority for CLI/API/future FW UI
configuration vs operational vs historical management state
native platform drift detection rather than silent import
stable object identity for automation/history
stale-candidate/concurrency protection direction
separate local-console recovery and remote-management trust paths
Net-Hunter investigation UI does not grant FW configuration authority
domain-specific health rather than one appliance health bit
capture/durability/forwarding/Hunter/journal/time/trust/HA/storage health separation
metrics vs authoritative journal separation
stateful alert lifecycle and maintenance/suppression semantics
explicit query-coverage/backlog observability
SNMPv3/syslog/structured API external-monitoring direction
no automatic packet/config/support telemetry to ISS
structured sensitivity-aware support bundles
explicit PCAP/support-artifact provenance
no permanent vendor support backdoor or hidden remote tunnel
customer-initiated/time-bounded/scoped future remote support
exceptional native engineering access with drift/qualification validation
hardware vendor preferred/supported OOB management below appliance layer
Stronghold qualification remains separate from vendor hardware/firmware supportability
selective TLS/application proxying as future inspection path
inspection remains encrypted on both sides of FW proxy
FW decrypted inspection payload is volatile-only and not intentionally persisted
purpose-specific mutually authenticated encrypted IDS transport
HISTORY and INSPECTION transport authority/identity/resource separation
future isolated Net-Hunter IDS/Inspection Jail
Hunter-host-controlled encrypted-at-rest inspection dataset
findings-only default inspection-retention direction
bounded/full decrypted retention requires explicit later policy
no default retention of TLS session secrets
explicit inspection coverage/bypass/failure truth
normal IDS overload does not silently backpressure production forwarding
mTLS/pinned applications bypass unless explicitly supported
QUIC/HTTP3 handling remains explicit rather than hidden
NIST SP 800-207 PE/PA/PEP direction for zero-trust remote access
WireGuard as zero-trust remote-access secure transport/data plane
Stronghold Access Session as authorization/session authority
WireGuard peer/AllowedIPs separated from Stronghold resource authorization
first-class grant/deny/revoke lifecycle
Stronghold Access Agent direction
L2TPv3 protected by IPsec for site-to-site / branch-office tunneling
site/tunnel identity separated from tunnel reachability
site-to-site tunnel state remains subject to normal Stronghold policy
```

## Later HA Qualification

Later HA gates must separately validate:

```text
forwarding failover
configuration synchronization
software/protocol compatibility
state/session continuity
split-brain prevention / fencing
cluster virtual identity ownership
capture continuity / gaps
failed-node local-history reconciliation
journal truthfulness
platform-trust eligibility
```

Node performance is not cluster performance. Net-Hunter remains outside FW quorum/fencing.

## Later Appliance Lifecycle / Update Qualification

Freeze and validate:

```text
release identity/signing/key lifecycle
online/offline delivery
version/upgrade compatibility windows
configuration-schema migration/rollback
journal/runtime-state compatibility
HA protocol/state negotiation
standby-first rolling upgrade
stop-on-standby-update-failure
post-update capture/enforcement regression
standalone outage behavior
Hunter host/jail/application ordering
ZFS pool-feature authorization boundary
update/rollback journals
```

Install/boot success is never the sole definition of update success.

## Later Recovery / DR Qualification

Freeze and validate:

```text
FW configuration backup/restore
secret/identity recovery separation
privileged Appliance-ID recovery
replacement-hardware NIC remapping/qualification
new-credential issuance/rebinding
HA failed-node replacement
failed-node offline PCAP recovery
Hunter host rebuild with surviving pools
complete Hunter/site-loss recovery source
journal-head divergence detection
hold/retention recovery
RECOVERY → VALIDATING → READY state transitions
destructive retention enabled last
permanent gap representation
backup/restorability verification
```

## Later Encryption / Key-Management Qualification

Freeze and validate:

```text
Stronghold cryptographic profile
FW encrypted-block implementation
Hunter ZFS dataset encryption hierarchy
TPM/HSM support boundaries
secret-store representation
per-purpose key hierarchy
recovery-key/package authority
key backup/recovery tests
rotation/re-wrapping behavior
compromise response
crypto-shred authorization
journaled key lifecycle
```

Encryption must not make authoritative history unrecoverable solely because one motherboard/TPM failed.

## Later Journal Integrity Qualification

Freeze and validate:

```text
canonical exact-byte entry representation
Journal ID / epoch / sequence contract
hash-linked entry formula
record framing / crash-tail handling
durable batch commit boundary
journal segment format
checkpoint content/cadence
journal signing credential purpose
signing-key rotation/compromise behavior
FW→Hunter verification / ACK
external head anchoring
continuity-gap / new-epoch recovery
retention tombstone/checkpoint lineage
verification status reporting
```

Checkpoint signing must not become a reason to drop observable traffic.

## Later Platform Trust / Hardware Qualification

Freeze and validate:

```text
supported Secure Boot posture
measured-boot/TPM behavior
attestation scope if adopted
platform-trust states
HA promotion consequences
hardware support/profile format
CPU/RAM/ECC requirements
NIC/driver/firmware matrix
AF_XDP/XDP driver and operating-mode matrix
AF_XDP queue/RSS/UMEM/affinity qualification
PCIe / NUMA qualification
NVMe sustained-write/endurance profile
feature-specific performance profiles
HA pair qualification
Hunter ingest/query/rebuild/degraded tests
release-to-hardware compatibility matrix
```

Stronghold performance claims apply to qualified profiles, not arbitrary hardware.

## Later Net-Hunter Records / Search / Reprocessing Qualification

The governing architecture is `docs/NET-HUNTER-RECORDS.md`.

Freeze and validate:

```text
segment-catalog authority and lifecycle
traffic/record catalog boundaries
flow/session identity and aggregation
Layer-2/control-plane derived record families
historical MAC/IP/name/VLAN relationship representation
derived provenance classes and lineage
source PCAP/journal locators
query/search pivot model
query coverage/completeness reporting
unknown/unsupported/malformed traffic discoverability
index rebuild / degraded-index behavior
segment-catalog fallback when indexes are incomplete
processing-generation / decoder-version lineage
targeted reprocessing scopes
full derived-generation rebuild/switch behavior
Hunter Processing Journal integration
transfer vs ingest vs processing vs reprocessing vs index backlog states
historical configuration-generation correlation
traffic-result → authoritative-PCAP pivot
export provenance
resource priority between current ingest, query, and historical reprocessing
```

Database/index technology, physical schemas, partitioning, exact Flow ID contracts, query language, and UI/API representation remain implementation choices to be selected after scale and workload measurements.

An incomplete/rebuilding index must never produce an unqualified `no results` claim.

## Later Management Plane Qualification

The governing architecture is `docs/MANAGEMENT-PLANE.md`.

Freeze and validate:

```text
single management authority used by CLI/API/future FW UI
candidate/validate/diff/commit/rollback consistency
configuration vs operational vs historical state representation
local console / break-glass boundaries
remote-management service exposure
RBAC parity across management surfaces
stable object identity / automation contracts
API versioning boundaries
stale-candidate/concurrency behavior
secret read/write behavior
native platform drift detection/reconciliation
diagnostic vs test vs repair semantics
administrative attribution/origin-surface journaling
Hunter UI vs FW management authority separation
```

A native OS modification is not a Stronghold commit, and an API endpoint does not bypass Stronghold validation/authorization.

## Later Observability / Alerting Qualification

The governing architecture is `docs/OBSERVABILITY.md`.

Freeze and validate:

```text
health-domain/state model
overall derived appliance summary rules
capture/drop/durability health
storage device vs pressure health
Hunter backlog dimensions
query-coverage health
journal/checkpoint/anchor health
time and trust health
HA multidimensional health
WAN destination/service health
policy/routing generation/runtime mismatch reporting
metrics vs journal boundaries
alert lifecycle / acknowledgement / resolution
maintenance/suppression behavior
capacity forecasting
syslog delivery state
SNMPv3 object/notification model
structured API health model
future webhook adapters
diagnostic-log retention
```

A running process/interface link does not prove subsystem/capture health, and alert resolution never erases a historical gap.

## Later Support / Diagnostics / Vendor OOB Qualification

The governing architecture is `docs/SUPPORT-DIAGNOSTICS.md`.

Freeze and validate:

```text
normal structured diagnostics
diagnose/test/repair boundaries
support-bundle manifest
sensitivity/redaction policy
explicit PCAP inclusion and provenance
support bundle encryption/integrity/export
no automatic ISS upload
support RBAC
future remote-support initiation/scope/expiry/revocation
support identity attribution
native engineering/break-glass access
drift/qualification validation after native modification
temporary debug limits
crash/core/memory handling
support-artifact retention
vendor OOB/BMC management preference
Stronghold vs BMC trust-domain separation
vendor-supported vs Stronghold-qualified firmware/hardware state
recovery-console use through vendor OOB where appropriate
```

Stronghold does not embed a permanent vendor backdoor or replace the qualified hardware vendor's supported out-of-band management platform.

## Later IDS / TLS Inspection Qualification

The governing future architecture is `docs/IDS-INSPECTION.md`.

**Implementation remains deferred.** Before any IDS/TLS-inspection implementation is promoted into a build phase, freeze and validate:

```text
selective inspection-policy model
proxy implementation and supported protocols
customer inspection-CA enrollment / recovery model
origin-certificate validation behavior
pinned/mTLS application bypass behavior
QUIC/HTTP3 behavior
FW volatile-only plaintext handling
swap/core/debug/support plaintext-exposure controls
purpose-specific IDS/inspection transport credential profile
explicit peer authorization
encrypted inspection transport framing/protocol
HISTORY vs INSPECTION queue/resource separation
inspection feed accounting / gap representation
future IDS/Inspection Jail isolation
Hunter encrypted inspection dataset / key hierarchy
findings schema and provenance
findings-only vs bounded-context/full-session retention classes
independent IDS retention/hold/destruction behavior
ruleset/engine generation identity
inspection-coverage reporting
normal overload/failure behavior
any explicit MUST_INSPECT/fail-closed policy behavior
future IPS enforcement boundary
performance/capacity qualification
```

Stronghold FW never intentionally persists decrypted inspection payload at rest, and there is no plaintext fallback from the encrypted IDS transport.

Authoritative encrypted-wire PCAP remains the packet-history authority. IDS findings are derived interpretation.

## Later Secure Access Qualification

The governing future architecture is `docs/SECURE-ACCESS.md`.

**Implementation remains deferred.** Stronghold secure access is architecturally separated into zero-trust remote access and site-to-site / branch-office tunneling.

Before zero-trust remote access is promoted into an implementation phase, freeze and validate:

```text
NIST SP 800-207 PE / PA / PEP responsibility boundaries
Stronghold Access Session schema/lifecycle
user/device identity model
MFA model
posture model and truth boundaries
Stronghold Access Agent platform/enrollment/update model
WireGuard key/session lifecycle
WireGuard AllowedIPs vs Stronghold resource-policy boundary
tunnel address allocation
resource authorization model
grant/deny/revoke and reauthorization behavior
control-plane outage behavior
PE/PA/PEP performance/scaling
HA behavior
capture/journal representation
Hunter correlation
management/API/RBAC
health/alerting
```

Before site-to-site / branch-office VPN is promoted into an implementation phase, freeze and validate:

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

WireGuard peer authentication never substitutes for Stronghold user/device/resource authorization. L2TPv3/IPsec tunnel establishment never substitutes for Stronghold traffic authorization.

## Explicitly Deferred — Secure Access Implementation

**Zero-trust remote-access and site-to-site / branch-office secure-access implementation are shelved/deferred.**

The future architecture is defined at concept level in `docs/SECURE-ACCESS.md`, but no production Access Agent, complete PE/PA/PEP implementation, WireGuard lifecycle implementation, posture engine, L2TPv3/IPsec implementation, interoperability profile, or complete HA/failover contract is selected for implementation now.

The defined architecture does not pull secure access into Phase 0 and does not weaken the authoritative physical-interface capture model.

## Explicitly Deferred — IDS/IPS Implementation

**IDS/IPS implementation is shelved/deferred.**

The future TLS/application inspection architecture is now defined at concept level in `docs/IDS-INSPECTION.md`, but no IDS/IPS engine, detection-ruleset format, complete proxy implementation, QUIC implementation, complete fail-open/fail-closed contract, or IPS enforcement mechanism is selected for implementation now.

The defined architecture does not pull IDS/IPS into Phase 0 and does not weaken the authoritative physical-interface capture model.

## Remaining Architecture Requiring Deliberate Freezing

Major areas still not fully frozen include:

```text
exact cryptographic algorithm/module profile
exact recovery-key authority/mechanics
exact journal canonical encoding/signature algorithms
exact external journal witness/anchor design, if adopted
exact HA fencing/election implementation
exact supported hardware profiles
Net-Hunter off-system backup/replication/archive architecture
exact Net-Hunter database/index technology and physical schemas
exact Net-Hunter query language/API/partitioning strategy
installation / factory provisioning / first-boot bootstrap architecture
later dynamic-routing implementation contracts
exact future Layer-7 boundaries outside defined inspection principles
secure-access implementation — deferred
IDS/IPS implementation — deferred
```

The exact implementation phase sequence for later capabilities remains intentionally unfrozen until the architecture is sufficiently complete.

No later capability is pulled into Phase 0 without explicit approval.