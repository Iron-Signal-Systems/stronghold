# Stronghold Architecture

## Purpose

This document records the current high-level architecture for the complete Stronghold system while preserving the capture-first engineering sequence already established for implementation.

> **Stronghold is an enforcement system that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.**

Stronghold consists of:

- **Stronghold FW** — live observation, preservation, authorization, forwarding, enforcement, journaling, local backlog, and optional active/standby HA.
- **Stronghold Net-Hunter** — verified historical ingest, durable preservation, processing/reprocessing, correlation, hunt/query, export, retention/holds, and isolated FW configuration backup.

```text
OBSERVE → RECORD → ENFORCE → REMEMBER → HUNT
```

> **Observe the truth. Preserve the history. Explain the decision.**

## Truth Model

```text
WHAT WAS PRESENTED
    authoritative physical-interface observation / PCAPNG

WHAT STRONGHOLD DID
    authoritative operational and decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation, correlation, and enrichment
```

### Observation survives interpretation

If Stronghold preserves a frame that it cannot decode today, that packet remains authoritative history. Later decoders/correlation may produce new derived interpretation without rewriting the source observation.

Missing interpretation must never erase observation, and missing history must never be presented as proof of absence.

Examples:

```text
OBSERVED + DECODER_NOT_AVAILABLE
OBSERVED + POLICY_DECISION_RECORD_INCOMPLETE + CAPTURE_REMAINS_AVAILABLE
CAPTURE_GAP reason=KERNEL_DROP
DURABLE_CAPTURE=FAILED reason=STORAGE_EXHAUSTED
TIME_CONFIDENCE=DEGRADED
```

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **A packet does not earn routing, NAT, or deeper forwarding work merely because it is technically routable.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

> **Stronghold owns the operating-system and appliance release lifecycle.**

> **Record what the system can actually establish, preserve the source of that truth, and never manufacture certainty beyond it.**

## Common Truth Boundaries

```text
route available                 != route authorized
certificate valid               != peer authorized
packet not decoded              != packet not observed
no journal record               != event did not occur
timestamp precision             != timestamp accuracy
policy allowed                  != forwarding succeeded
object expired                  != destruction authorized
bytes received                  != history durably committed
archived                        != destroyed
new interpretation              != new historical observation
forwarding HA succeeded         != capture continuity guaranteed
virtual cluster identity        != physical capture identity
software installed              != Stronghold update validated
configuration restored          != appliance validated
backup written                  != backup proven restorable
ZFS redundancy                  != disaster recovery
disk encrypted                  != secrets safely designed
TPM protected                   != disaster recoverable
cluster member                  != shared local disk key
key rotated                     != past exposure undone
entry written                   != journal entry durably committed
hash chain valid                != checkpoint externally anchored
signed journal checkpoint       != physically immutable storage
new journal epoch               != uninterrupted journal continuity
system booted                   != approved system state
Secure Boot enabled             != measured boot verified
measured boot available         != attestation performed
Appliance ID                    != TPM identity
hardware detected               != hardware qualified
hardware compatible             != hardware supported
link speed                      != validated dataplane rate
capture throughput              != full-feature firewall throughput
node performance                != HA cluster performance
storage capacity                != sustainable ingest capacity
peak benchmark                  != sustained appliance performance
```

## Capture Invariant #1 — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Physical-interface frame observation is a product invariant of the enforcement appliance itself.

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline. Protocol recognition is not a prerequisite for capture.

Stronghold remains truthful about NIC filtering, driver behavior, VLAN/checksum offloads, aggregation/coalescing, hardware timestamping, capture-ring/kernel drops, storage failures, and any condition that alters or prevents observation.

## System Topology

```text
PRODUCTION NETWORK
        │
        ▼
┌─────────────────────────────┐
│        STRONGHOLD FW        │
│ Observe / Preserve          │
│ Authorize / Bridge / Route  │
│ NAT / Enforce / Journal     │
│ Arch Linux                  │
│ Btrfs OS                    │
│ XFS NVMe HOT                │
│ XFS SSD WARM/BACKLOG        │
└──────────────┬──────────────┘
               │ dedicated HISTORY network
               │ mTLS + explicit peer authorization
               ▼
┌─────────────────────────────┐
│    STRONGHOLD NET-HUNTER    │
│ Receive / Verify / Commit   │
│ Preserve / Reprocess        │
│ Correlate / Hunt / Query    │
│ Export / Retain / Archive   │
│ FreeBSD / ZFS / Jails       │
└─────────────────────────────┘
```

Both appliances also use explicit MANAGEMENT roles/interfaces. Optional FW HA uses a distinct HA role/interface.

# Stronghold FW

## Platform Direction

```text
Platform:          Arch Linux
Architecture:      x86_64
Administration:    CLI first
NICs:              qualified physical PCIe Ethernet
Initial capture:   AF_PACKET / TPACKET_V3
Capture format:    PCAPNG
OS storage:        dedicated SSD / Btrfs direction
HOT capture:       NVMe / XFS direction
WARM/backlog:      SSD / XFS direction
1 GbE:             current qualification target
10 GbE:            primary production target
40 GbE:            future target only
```

## Networking Model

Stronghold does not require one global forwarding mode.

```text
Layer-2 transparent bridge
Layer-3 IPv4/IPv6 routed firewall
router-on-a-stick over 802.1Q trunks
hybrid Layer-2/Layer-3 deployment
```

VLANs are first-class objects. Expected object families include physical/logical interfaces, VLANs, bridge domains, zones, hosts/networks/groups, services/groups, FQDNs/groups, routes, security policy, NAT policy, WAN preference, HA cluster/nodes, and virtual forwarding identities.

Current interface roles:

```text
PRODUCTION
WAN
MANAGEMENT
HISTORY
HA
```

MANAGEMENT, HISTORY, and HA are not ordinary production transit. HISTORY is not HA sync; HA is not Hunter transfer; neither is a normal interactive admin path.

## Policy and Authorization

Stronghold is explicit-allow/default-deny.

Rules use dense mutable integer positions, lowest-to-highest, first match wins. **Policy ID is stable reference identity only and never priority/order.**

```text
PACKET / FRAME ARRIVES
        │
        ├──────────────► CAPTURE / OBSERVE / PRESERVE
        ▼
ESTABLISH REQUIRED POLICY FACTS
        ▼
EARLY AUTHORIZATION
        ├── explicit DENY ───────────────► DROP
        ├── no permitting match ─────────► DROP
        └── authorized to proceed
                    ↓
              BRIDGE / ROUTE
                    ↓
          WAN / NAT / STATE /
       REQUIRED DEEPER PROCESSING
                    ↓
               FINAL EGRESS
                    ↓
           JOURNAL ACTUAL RESULT
```

A route/FIB lookup may be technically required to establish a fact, but successful lookup never constitutes authorization.

`NOT_PERFORMED` is explicit when later pipeline work never occurred.

## Routing

Only authorized traffic enters normal routing.

```text
1. longest-prefix match
2. equal-prefix route-source preference:
       STATIC
       CONNECTED
       DEFAULT
       DYNAMIC
3. health / availability
4. metric
5. deterministic tie-break
```

A policy ALLOW with no route is a routing failure/final DROP, not a policy denial. VRF remains future-compatible, not initial.

## NAT

NAT never grants permission. Initial direction includes masquerade, static SNAT, DNAT/port forwarding, 1:1 NAT, and no-NAT. Original and translated tuples remain separate authoritative decision facts.

## Multi-WAN

Stronghold uses policy-defined WAN eligibility/preference, not generic traffic spraying. A physically available WAN is not automatically authorized. Existing sessions normally remain bound to established WAN/NAT identity.

## Configuration Management

```text
RUNNING
  ↓
CANDIDATE
  ↓ validate
SHOW DIFF
  ↓
COMMIT
  ↓
NEW MONOTONIC GENERATION
```

Rollback restores older content by creating a new generation. Risky remote changes support commit-confirmed protection.

Configuration generation, configuration schema, software release, journal schema, and HA protocol/state format are separate identities.

# Stronghold FW High Availability

## Initial Model

Optional **active/standby** only. Active/active is not the initial HA model.

Each node retains its own:

```text
Appliance ID
physical NIC/MAC identity
management/history/HA identity
node certificates
local storage
clock state
source journals
capture provenance
```

The pair has a stable Cluster ID. Cluster-owned virtual IP/MAC forwarding identity belongs only to the ACTIVE node.

> **Virtual network identity is forwarding identity, not capture identity.**

Routed/VLAN HA uses cluster-owned gateway identity. Transparent L2 HA uses explicit bridge ownership so only one path forwards.

## Cluster vs Node Configuration

Cluster configuration includes policy, VLANs/zones/bridge domains, routes, NAT, WAN preference, virtual identities, and cluster settings.

Node-local configuration includes Appliance ID, management/history/HA addressing, physical NIC/PCI mapping, local storage mapping, and node certificate identity.

Normal cluster commits validate cluster semantics and each node's hardware/local context before coordinated activation. A stale standby is a degraded failover target.

## State Synchronization

May include firewall sessions, NAT mappings, timers, selected WAN, route/session binding, and justified runtime identity state.

Expected health concepts:

```text
IN_SYNC
MINOR_LAG
DEGRADED
OUT_OF_SYNC
UNKNOWN
```

Heartbeat/control and bulk synchronization are logically separate. Heartbeat/control is lightweight and reserved; bulk sync may throttle/lag. Capture remains the highest live-data priority.

## Promotion and Split-Brain

> **Loss of peer communication is not, by itself, proof that the peer is dead.**

> **A node may claim ACTIVE cluster identity only after satisfying the fencing/election contract.**

Promotion evaluates local health, network readiness, peer evidence, fencing/election safety, config state, software/protocol compatibility, state-sync condition, and later platform-trust state.

Net-Hunter is not an HA witness/quorum dependency.

Successful forwarding failover does not imply session continuity or zero capture gap.

# Appliance Lifecycle and Updates

Stronghold FW is an appliance built on Arch Linux, not a general-purpose customer-managed rolling host. Routine unrestricted `pacman -Syu` is outside the supported lifecycle.

Stronghold-qualified releases bind/qualify relevant combinations of:

```text
Stronghold software
Linux kernel
NIC drivers / managed firmware
nftables / netfilter
capture stack
required system libraries
configuration schema
journal/schema versions
HA protocol/state format
```

Update material must be cryptographically verified before activation. Signed offline bundles are required for restricted deployments.

```text
SIGNED / VERIFIED RELEASE
        ↓
PREFLIGHT
        ↓
COMPATIBILITY CHECK
        ↓
CHECKPOINT / SNAPSHOT WHERE APPLICABLE
        ↓
INSTALL
        ↓
REBOOT IF REQUIRED
        ↓
STRONGHOLD VALIDATION
        ↓
READY / STANDBY_READY / ACTIVE_READY
```

Install success and boot success are not equivalent to validation success. Kernel/NIC/firmware/netfilter/capture/storage changes require capture/enforcement regression qualification appropriate to the change.

Configuration/schema migrations are controlled transitions with recoverable source state until target validation completes.

Btrfs snapshots may assist rollback, but rollback must account for boot/kernel state, software version, config schema, journal/runtime formats, HA compatibility, and managed firmware implications.

## HA Rolling Update

```text
FW-A ACTIVE
FW-B STANDBY
      ↓
update FW-B
      ↓
validate FW-B
      ↓
controlled failover
      ↓
FW-B ACTIVE
FW-A STANDBY
      ↓
validate production state
      ↓
update FW-A
      ↓
validate cluster
```

Mixed-version operation is temporary and allowed only inside an explicitly supported compatibility window.

> **A failed standby update stops the rolling process. Stronghold must not automatically risk the remaining healthy active node.**

A standalone FW update that requires reboot/service interruption has a real forwarding/capture outage and reports it truthfully.

# Recovery and Disaster Recovery

## Recovery Classes

Stronghold separates:

```text
REBUILDABLE PLATFORM
RECOVERABLE CONFIGURATION
APPLIANCE / TRUST IDENTITY
AUTHORITATIVE HISTORY
DERIVED STATE
TRANSIENT RUNTIME STATE
```

A recovery workflow does not treat an appliance as one opaque disk image.

## FW Recovery

FW committed configuration generations are versioned in the isolated Net-Hunter FW Configuration Backup Jail. Configuration backup is separate from protected secret/private-key recovery.

Complete hardware replacement enters an explicit `RECOVERY` workflow:

```text
install qualified Stronghold release
      ↓
authorized recovery of logical Appliance ID
      ↓
restore/synchronize configuration
      ↓
generate/re-establish cryptographic credentials
      ↓
map and validate physical interfaces
      ↓
validate capture / policy / routing / NAT / HA as applicable
      ↓
READY / STANDBY_READY / ACTIVE_READY
```

Restoring an Appliance ID is a privileged identity-recovery action, not ordinary configuration import. Replacement hardware normally generates new private keys/certificates and rebinds trust while preserving the logical Appliance ID.

Physical NIC mapping must be explicitly validated; old Linux interface naming is not blindly trusted. Runtime conntrack/NAT/session state is not restored from stale backup. An HA replacement node synchronizes current runtime state from the active peer where applicable.

If failed-node PCAP storage survives, recovery begins read-only where practical and preserves original source Appliance ID, observation time, segment identity, and lineage. Hunter records the later offline recovery fact separately.

## Net-Hunter Recovery

Authoritative Hunter state includes PCAP, FW source journals, Hunter Processing Journal, configuration backups, integrity/lineage data, retention/hold/destruction state, and trust state. Derived indexes/correlations are rebuildable from authoritative source data.

ZFS RAID/checksums/snapshots are protection mechanisms, not disaster recovery.

A host-boot-device failure with intact pools should permit qualified host reinstall, pool import, dataset/jail validation, and derived-state rebuild without rewriting authoritative history.

Complete site/storage loss requires off-system backup/replication/archive architecture later; RAIDZ2 alone is not that architecture.

Hunter Appliance ID recovery is explicit and privileged. Replacement hardware normally generates new private credentials while preserving logical identity only when authorized.

After significant Hunter recovery:

```text
RECOVERY
  ↓
platform/identity validation
  ↓
authoritative storage validation
  ↓
journal/lineage validation
  ↓
trust authorization validation
  ↓
hold-state validation
  ↓
retention-state validation
  ↓
derived/index rebuild
  ↓
ingest/query enable
  ↓
destructive retention enable LAST
```

A missing authoritative interval remains a permanent explicit history gap. Recovery never silently heals it.

# At-Rest Encryption and Key Management

## Protection Classes

```text
AUTHORITATIVE PACKET HISTORY
JOURNALS / CONFIGURATION HISTORY
SECRETS / PRIVATE CREDENTIAL MATERIAL
DERIVED / TEMPORARY DATA
```

Stronghold uses purpose-separated encryption domains rather than one deployment-wide master key.

## FW Direction

Preferred direction is encrypted block devices below the filesystem:

```text
OS SSD      encrypted block layer → Btrfs
HOT NVMe    encrypted block layer → XFS
WARM SSD    encrypted block layer → XFS
```

This keeps application packet writing simple and capture-first. Exact Linux crypto profile remains to be frozen.

TPM 2.0 is an anticipated normal-key-protection mechanism. TPM-bound material is not the sole recovery path for authoritative history.

Node-local disk keys remain node-local even in an HA cluster.

## Hunter Direction

Net-Hunter favors ZFS-native encrypted datasets aligned with host/jail authority. Separate encryption roots/keys may protect authoritative history, journals, derived/index state, FW configuration backups, and UI workspace.

The FreeBSD host retains key-management authority; jails receive only datasets and access required for their role.

## Secrets and Private Keys

Stronghold configuration references secret identities rather than embedding plaintext credentials throughout general config. Appliance identity private keys should be non-exportable where supported.

Hardware replacement normally generates new private credentials and re-establishes authorization rather than cloning old identity keys.

## Recovery Authority

Recovery authority is separate from normal appliance operation, ordinary admin authentication, and history-hunt privileges. Recovery material is per-appliance/system scope rather than one universal customer decryption key.

Key backup must be verified; `key backup created` is not `key recovery proven`.

Routine key rotation should not inherently require rewriting all historical PCAP. Key-wrapping/key-encryption rotation and full data re-encryption are distinct operations.

Key compromise events preserve the fact that earlier confidentiality may have been affected; rotation does not retroactively undo exposure.

Crypto-shredding, if ever supported, is a high-risk destructive action subject to explicit privilege, hold evaluation, reason, journaling, and verification.

# Journal Advancement and Tamper Evidence

## Independent Journal Streams

Stronghold maintains separate authoritative streams:

```text
Administrative Journal
System / Health Journal
Traffic Decision Journal
Trust / Identity Journal
Time / Clock Journal
Hunter Processing Journal
```

Each stream has its own stable Journal ID, origin Appliance ID, domain, epoch, monotonic local sequence, and hash-linked committed entries.

Wall-clock time does not establish journal order; local journal sequence does. Clock state/timestamp remain recorded facts.

## Canonical Integrity Representation

Journal hashing/signing operates over a canonical exact-byte representation. Serialization ambiguity is not permitted inside the integrity contract. Exact canonical encoding remains to be frozen before implementation.

Conceptually:

```text
Hn = HASH(journal_id || epoch || sequence || previous_hash || canonical_entry_body)
```

## Commit and Batching

```text
entry built
   ↓
framed/written
   ↓
required durability boundary
   ↓
authoritative journal head advances
   ↓
COMMITTED
```

`write()` success is not journal durability. High-volume journals may commit durable batches while preserving per-entry sequence/hash linkage. Uncommitted crash tails are not represented as committed history.

## Checkpoints and Finalized Journal Segments

Asymmetric signing is applied to periodic checkpoints/finalized journal segments rather than every event.

A checkpoint binds at least the Journal ID, epoch, covered sequence range, final head hash, predecessor checkpoint/segment identity, origin Appliance ID, and signing identity.

Checkpoint triggers may later include count, time, size, planned shutdown, update, HA transition, manual checkpoint, or segment finalization. Exact cadence is performance-qualified.

Journal signing credentials are purpose-separated from TLS/history/HA credentials. Digital signatures are preferred for long-term third-party verification because public-key verifiers cannot forge source checkpoints.

## Epochs and Recovery

Normal reboot continues the existing journal when the last durable head can be verified.

A catastrophic continuity break uses a new epoch and explicit gap/recovery fact. New epoch is not represented as uninterrupted history.

Crash recovery finds the last valid durable committed boundary, verifies chain/framing/checkpoint state, preserves/quarantines incomplete tail as required, and records recovery outcome.

## FW → Hunter Verification

Finalized FW journal segments/checkpoints transfer to Hunter. Hunter verifies segment integrity, hash linkage, signed checkpoint, expected predecessor, signing-key validity, and durable commit before ACK.

Hunter preserves source FW journals without rewriting them and creates separate Hunter Processing Journal facts for receive/verify/commit/reprocess activity.

Hunter may preserve externally known FW journal heads. Hunter outage does not stop FW journal advancement; backlog catches up later and continuity is verified. A mismatch is not silently accepted.

## Retention of Journal Content

Authorized retention/destruction may remove old detailed journal content while preserving sufficient segment/checkpoint/tombstone lineage to establish that the range existed, its integrity identity, and its authorized destruction.

Deletion never rewrites adjacent chains to pretend the removed range never existed.

Stronghold describes the journals as **append-oriented and cryptographically tamper-evident**. Stronger physical-immutability claims require a later WORM/external-witness design.

# Platform Trust and Secure Boot Posture

Stronghold distinguishes:

```text
logical Appliance ID
hardware/root-of-trust identity
certificate identity
platform trust state
```

Secure Boot, measured boot, and attestation are separate capabilities.

Preferred FW direction:

```text
UEFI firmware
  ↓
Secure Boot
  ↓
approved bootloader
  ↓
approved kernel/initramfs
  ↓
Stronghold appliance state
```

TPM may support measured boot, sealed volume-unlock material, non-exportable keys, hardware identity binding, and future attestation. TPM is not the sole recovery authority.

Hunter aims for equivalent trust properties appropriate to supported FreeBSD/hardware capabilities without prematurely freezing unsupported implementation details.

A future platform-trust state may use concepts such as:

```text
VERIFIED
DEGRADED
UNVERIFIED
MISMATCH
RECOVERY
```

An HA standby with unacceptable platform-trust state is not silently considered a normal promotion target.

Update-bundle verification and post-boot platform validation are separate checks.

# Hardware Qualification and Supported Appliance Profiles

## Qualification States

```text
DETECTED
COMPATIBLE
QUALIFIED
SUPPORTED
```

Stronghold support/performance claims apply to qualified appliance profiles, not arbitrary hardware capable of running the software.

A profile binds at least:

```text
Stronghold release
CPU / architecture
RAM / ECC requirement as applicable
NIC model / driver / firmware
PCIe generation/lanes/topology
NUMA placement where applicable
HOT/WARM storage class
platform trust/firmware state
enabled feature set
```

## NIC Qualification

Because capture visibility is a product invariant, NIC qualification includes:

```text
driver stability
multi-queue / RSS
VLAN representation
offload behavior
promiscuous/filter behavior
ring/drop behavior
MTU/jumbo behavior
timestamp behavior
link/reset behavior
firmware revision
```

Wireshark/dumpcap remains the reference visibility comparison under equivalent supported conditions.

## Dataplane Qualification

Qualification measures both Gbps and packets/sec and includes representative packet sizes, mixed traffic, bidirectional traffic, flow counts, new-session bursts, and sustained steady state.

Feature profiles remain separate, for example:

```text
CAPTURE ONLY
CAPTURE + L2
CAPTURE + L3
CAPTURE + STATEFUL FIREWALL
CAPTURE + FIREWALL + NAT
CAPTURE + FIREWALL + MULTI-WAN
CAPTURE + FIREWALL + HA STATE SYNC
```

A capture-only result is not reused as a full-feature firewall claim.

Loss accounting separates NIC, ring/kernel, writer/storage, forwarding, and intentional policy drops where measurable.

## Storage Qualification

HOT NVMe qualification emphasizes sustained throughput, latency, queue behavior, thermals/throttling, power-loss behavior, endurance, firmware, filesystem behavior, and long-duration performance rather than short peak benchmarks.

RAM is buffering/state capacity, not durable history.

## HA Qualification

Node benchmarks do not establish cluster performance. HA qualification measures peer-loss detection, fencing/election, promotion, virtual identity convergence, forwarding restoration, session survival, state-sync lag, capture interruption/gap, and journal transition truthfulness separately.

## Net-Hunter Qualification

Hunter qualification separates capacity from sustainable ingest, verification/hash rate, journal ingest, indexing, query, PCAP retrieval, reprocessing, retention/expiration, scrub, resilver, and degraded-storage behavior.

ECC remains the required direction for Net-Hunter. FW ECC is strongly preferred for production and may become profile-required when exact supported hardware is frozen.

Development hardware, controlled qualification hardware, and supported production profiles are distinct categories.

# Packet History, Handoff, and Net-Hunter

## FW Local Storage

```text
RAM buffer
    ↓
NVMe / XFS HOT
    ↓
SSD / XFS WARM/BACKLOG
    ↓
dedicated HISTORY network
    ↓
Net-Hunter
```

PCAP storage remains separate from the OS filesystem. Hunter outage creates explicit backlog while local capacity remains.

## Verified Handoff

```text
FW finalizes segment/source journal batch
       ↓
FW integrity metadata
       ↓
mTLS transfer to explicitly authorized Hunter
       ↓
Hunter receive/finalize
       ↓
independent verification
       ↓
durable commit
       ↓
Hunter Processing Journal
       ↓
verified ACK
       ↓
FW acknowledgement state
```

Hunter never ACKs unverified/uncommitted history.

## Net-Hunter Jails

1. **PCAP Data Ingest Jail** — authenticated/authorized receive, verification, commit, ACK.
2. **Record Processing Jail** — reads authoritative PCAP/source history; builds/rebuilds derived searchable records/indexes.
3. **External User Interface Jail** — hunt/query/correlation/view/export with authoritative history read-only.
4. **FW Configuration Backup Jail** — isolated versioned FW configuration history.

The host owns hardware, HBAs/raw disks, ZFS, host networking/PF, jail lifecycle, updates, and storage health.

## Net-Hunter Lifecycle

FreeBSD host, Stronghold host-management layer, jails, jail applications, derived schemas/indexes, and UI are related but independently controlled layers.

Derived indexes are rebuildable. Failed index migration must not rewrite authoritative PCAP/source journals.

A FreeBSD/ZFS software update does not automatically authorize enabling irreversible ZFS pool features. Pool-feature upgrades are separate explicit compatibility operations.

# Retention, Holds, Archive, and Destruction

PCAP and each journal domain have independent policy.

```text
CREATE
  ↓
ACTIVE
  ↓
RETAINED
  ↓
ELIGIBLE_FOR_EXPIRATION
  ↓
AUTHORIZED_DESTRUCTION
  ↓
DESTROYED
  ↓
DESTRUCTION JOURNALED
```

Holds override ordinary expiration. Hold scope favors reproducible facts such as appliance, time range, VLAN, zone, IP/host, MAC, segment, or journal range.

Manual destruction is distinct from automatic retention and requires explicit privileged authority and reason. Archive is not destruction.

FW pressure states include `NORMAL`, `HIGH`, `URGENT`, `CRITICAL`. Pressure may throttle secondary work, accelerate safe transfer/tiering, and advance already-authorized expiration. It never invents destructive authority.

By default, unacknowledged authoritative FW history is not automatically deleted merely to hide storage pressure. If storage exhaustion prevents durable capture, Stronghold reports the gap truthfully.

# Administrative Identity and Trust

Remote auth direction:

```text
Active Directory via LDAPS only
RADIUS
TACACS+
```

No plaintext LDAP and no LDAPS→LDAP downgrade. Protected local identity remains for installation/recovery/break-glass.

Authentication, authorization, and journaling remain separate. Important permissions include configuration view/edit/validate/commit/rollback, network/security/system administration, HA administration, appliance update administration, history hunt/view/export, retention/hold/destruction administration, and recovery authority.

Each appliance has a stable Appliance ID independent of hostname/IP/current certificate. History transport uses mTLS plus explicit peer authorization. Certificate validity does not equal authorization.

# Time and Clock

Authoritative wall-clock timestamps are UTC. Timezone is presentation only. Local advancing journal/order state remains independent of wall-clock correctness.

Clock-state concepts include `SYNCHRONIZED`, `HOLDOVER`, `UNSYNCHRONIZED`, `CLOCK_FAULT`.

> **Timestamp precision must never be presented as timestamp accuracy.**

Hunter preserves source observation time separately from receive/verify/commit/process times.

# Resource Priority

On FW:

```text
1. packet acquisition
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation/decision/journal append
5. HA heartbeat/control reserved lightweight capacity
6. journal checkpoint/finalization work
7. local tier movement/backlog
8. HA bulk state synchronization
9. transfer to Net-Hunter
10. compression where approved
11. deeper analytics outside live path
```

Exact scheduler/queue implementation remains future work. Checkpoint signing, bulk HA sync, update activity, and secondary processing must not become reasons to hide or sacrifice capture.

# Explicit Deferrals

**VPN is deferred. IDS/IPS is deferred.**

Neither is part of Phase 0, an early dataplane claim, or a dependency of current capture design. Stronghold preserves future compatibility by keeping authoritative PCAP and derived interpretation separate, but no VPN design, IDS/IPS engine, inline IPS behavior, TLS interception, detection ruleset contract, or fail-open/fail-closed policy is selected now.

# Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

A complete feature should preserve enough history/state to answer what was observed, known, unknown, decided, actually performed, not performed, failed, lost, and still recoverable.

# Current Scope

Implementation still begins with the **Phase 0 Traffic Observation Foundation**. The broader networking, enforcement, HA, recovery/DR, encryption/key-management, journal-integrity, platform-trust, appliance-update, and hardware-qualification architecture is recorded now so Phase 0 choices do not block the complete system.

Later implementation phase sequencing remains intentionally unfrozen while the complete product architecture is still being defined.
