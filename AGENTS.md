# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation work within the Stronghold repository. It is a behavioral contract and does not replace `docs/ARCHITECTURE.md`, `docs/NET-HUNTER-RECORDS.md`, or `docs/ROADMAP.md`.

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

> **Record what the system can actually establish, preserve the source of that truth, and never manufacture certainty beyond it.**

> **An incomplete index is not complete history.**

## Product Model

### Stronghold FW

```text
observe
preserve
authorize
bridge / route
NAT where configured
enforce
journal
maintain local history backlog
optionally participate in active/standby HA
transfer verified history to Net-Hunter
```

### Stronghold Net-Hunter

```text
receive
verify
durably commit
preserve
journal processing state
process / reprocess
correlate
hunt
query
export
preserve FW configuration backups
manage authorized retention / holds / archive lifecycle
```

Net-Hunter must never become a runtime dependency for FW capture, forwarding, NAT, enforcement, or HA quorum/fencing.

## Three Categories of Truth

Preserve the distinction between:

```text
WHAT WAS PRESENTED
    authoritative physical-interface PCAPNG

WHAT STRONGHOLD DID
    authoritative operational / decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation / correlation / enrichment
```

Derived data remains traceable to authoritative source data. Reprocessing may create new/superseding derived interpretation but MUST NOT rewrite original PCAP or source journals.

## Capture Rules

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the visibility reference.

Unknown EtherTypes, unknown IP protocols, malformed/vendor-specific traffic, and Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

RAM is buffering only, not durable capture.

Initial acquisition direction is AF_PACKET/TPACKET_V3 unless later measured/approved contracts justify another path.

## Authorization and Forwarding Rules

Stronghold is explicit-allow/default-deny.

```text
observe / preserve
    ↓
establish required policy facts
    ↓
authorize
    ├── deny / no allow → drop
    └── authorized to proceed
             ↓
       bridge / route
             ↓
 WAN / NAT / state / deeper work
             ↓
          final egress
```

Do not let route availability, NAT, FQDN association, WAN reachability, or HA membership grant security permission.

A route/FIB lookup may establish a needed fact but MUST NOT itself constitute authorization.

Policy rules use dense integer positions, lowest-to-highest, first match wins. Policy ID is stable identity only, never priority.

Preserve `NOT_PERFORMED` when later pipeline work did not occur.

## Networking Rules

Supported architecture includes Layer-2 transparent bridging, Layer-3 IPv4/IPv6 routing, router-on-a-stick over 802.1Q, and hybrid deployments.

VLAN is first-class and preserves Stronghold identity plus actual VLAN ID.

Current interface roles:

```text
PRODUCTION
WAN
MANAGEMENT
HISTORY
HA
```

MANAGEMENT, HISTORY, and HA are not production transit. HISTORY is not HA sync. HA is not Hunter transfer. Neither is a normal interactive admin path.

Routing is longest-prefix first, then equal-prefix source preference `STATIC`, `CONNECTED`, `DEFAULT`, `DYNAMIC`, then health, metric, deterministic tie-break.

NAT is separate from authorization. WAN eligibility is explicit per destination/service.

## Common Truth Separations

Never collapse these distinctions:

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
signed checkpoint               != physically immutable storage
new journal epoch               != uninterrupted history
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
database record                 != authoritative packet
no indexed result               != no traffic
index unavailable               != history unavailable
source history available        != index complete
reprocessing result             != original knowledge
timeline                        != authoritative journal
exported PCAP                   != original PCAP segment
current hostname association    != historical hostname association
direct observation              != correlated association
processing failed               != source history lost
```

## Failure-State Discipline

Preserve meaningful states such as:

```text
unknown
not_known
not_observed
not_verified
not_performed
not_present
not_applicable
degraded
unavailable
mismatch
failed
recovery_required
continuity_gap
processing_pending
index_incomplete
```

Do not infer success from absence of error. A later successful retry does not erase a failed operation.

## Journal Rules

Maintain independent append-oriented journals:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Each journal has separate Journal ID/domain, origin Appliance ID, epoch, monotonic local sequence, and hash-linked committed entries.

Wall-clock time is not local journal order.

Hash/signature calculation requires a canonical exact-byte representation. Do not hash ambiguous serializer output.

`write()` success is not journal commit. Authoritative head advances only after the required durability boundary.

High-volume journals may use durable batch commits while preserving per-entry sequence/hash linkage.

Periodic signed checkpoints/finalized journal segments authenticate heads; do not require expensive asymmetric signatures for every event.

Journal-signing credentials are purpose-separated from TLS/history/HA credentials.

Normal reboot continues an existing verified journal. Catastrophic continuity failure creates an explicit gap/new epoch rather than silently restarting the chain as though nothing happened.

Hunter verifies FW journal linkage/checkpoints independently and preserves source FW journals without rewriting them. Hunter Processing Journal is a separate authority.

Authorized journal retention may remove content only while preserving enough checkpoint/tombstone lineage to establish that the range existed and was deliberately destroyed.

Use the term **append-oriented and cryptographically tamper-evident** unless a later WORM/witness design actually justifies stronger immutability claims.

## Retention / Hold / Destruction Rules

PCAP and each journal domain have independent retention.

Age/threshold establishes eligibility, not destruction authority. Holds override normal expiration.

Manual destruction requires explicit privileged authority and reason. Archive is not destruction.

Storage pressure cannot silently create emergency deletion authority or cause Hunter to ACK history it has not durably committed.

Crypto-shredding, if ever supported, is a privileged destructive operation subject to hold evaluation and is not an ordinary cleanup shortcut.

## HA Rules

Initial HA is optional active/standby only.

Each node keeps its own Appliance ID, physical capture provenance, management/history/HA identities, storage, certificates, and local keys. The pair has a Cluster ID and cluster-owned virtual forwarding identity.

Virtual forwarding identity never replaces physical capture provenance.

Heartbeat/control and bulk state sync are separate traffic classes. Heartbeat/control must not be starved; bulk sync may lag. Capture remains highest live-data priority.

> **Loss of peer communication is not, by itself, proof that the peer is dead.**

> **A standby must not claim ACTIVE cluster identity until the fencing/election contract permits promotion.**

Promotion evaluates health, path readiness, peer evidence, fencing/election, config state, software/HA compatibility, state-sync state, and later platform-trust state.

Net-Hunter is not HA quorum/witness.

Successful forwarding failover does not justify a zero-capture-gap claim.

## Appliance Update Rules

Stronghold FW is an appliance built on Arch Linux. Do not design supported production operation around unrestricted customer `pacman -Syu`.

A qualified release binds/validates relevant Stronghold software, kernel, NIC driver/firmware, nftables/netfilter, capture stack, schemas, and HA formats.

Update material must be cryptographically verified. Preserve signed offline update support.

Install or boot success is not update validation. Kernel/NIC/netfilter/capture/storage changes require appropriate capture/enforcement regression validation.

Configuration/schema migrations preserve recoverable source state until target validation succeeds.

Btrfs snapshots may assist rollback but do not define rollback correctness.

HA rolling update sequence is standby-first. If standby update/validation fails, STOP. Do not automatically risk the remaining healthy active node.

Standalone FW updates that interrupt service create real forwarding/capture outages and must be reported truthfully.

FreeBSD/ZFS software update does not automatically authorize irreversible ZFS pool-feature enablement.

## Recovery / DR Rules

Distinguish rebuildable platform, configuration, identity/trust, authoritative history, derived state, and transient runtime state.

Configuration backup is not identity/private-key backup.

Recovering an Appliance ID onto replacement hardware is explicit privileged recovery, not ordinary import.

Replacement hardware normally generates new private credentials and re-establishes authorization while preserving logical Appliance ID only when approved.

Do not blindly restore physical NIC mappings; validate PCI/MAC/driver/capture capability before activation.

Do not restore stale conntrack/NAT/session state from backup. HA replacement nodes synchronize current runtime state from the active peer where applicable.

Recovered failed-node PCAP retains original source Appliance ID/observation time/segment lineage and is marked as later recovered history.

ZFS RAID/snapshots are not DR.

After major Hunter recovery, validate authoritative storage, journals/lineage, trust, holds, and retention before enabling destructive retention. Destructive operations come back last.

Missing authoritative history remains an explicit gap; never silently heal it.

## Encryption / Key Rules

Stronghold uses purpose-separated encryption domains for packet history, journals/config history, secrets/private credentials, and derived/temp data.

FW direction favors encrypted block devices beneath Btrfs/XFS so application capture remains simple and capture-first.

Hunter direction favors ZFS-native encrypted datasets aligned with host/jail authority.

TPM/HSM may protect normal key use but must not be the sole recovery path for authoritative history.

HA nodes do not share local disk keys merely because they share cluster state.

Private appliance identity keys should be non-exportable where supported. Replacement hardware normally generates new private credentials.

General configuration should reference secret IDs rather than repeat plaintext secrets.

Recovery authority is separate from normal administration and hunting.

Routine key rotation should not inherently require rewriting all historical PCAP. Key wrapping and full data re-encryption are distinct.

Key rotation does not prove previously protected data was never exposed.

## Platform Trust Rules

Distinguish logical Appliance ID, hardware/root-of-trust identity, certificate identity, and platform-trust state.

Secure Boot, measured boot, and attestation are separate capabilities and must not be conflated.

TPM-backed measured/sealed behavior is an anticipated FW direction but cannot be the only DR path.

A valid signed update bundle does not prove the resulting booted appliance is in an approved state.

An HA standby with unacceptable platform-trust state is not silently a normal promotion target.

Development, qualification, and supported production hardware are separate categories.

## Hardware Qualification Rules

Stronghold claims apply to qualified appliance profiles, not arbitrary hardware that boots.

Preserve distinctions between `DETECTED`, `COMPATIBLE`, `QUALIFIED`, and `SUPPORTED`.

Qualification binds release, CPU, NIC/driver/firmware, PCIe/NUMA topology, RAM/ECC, storage, platform trust, and enabled feature set.

NIC qualification must cover visibility-affecting behavior: RSS/multi-queue, VLAN/offload representation, filtering/promiscuous behavior, ring/drop behavior, MTU, timestamps, reset behavior, and firmware.

Performance testing must include bandwidth **and** PPS, multiple packet sizes, sustained duration, traffic mix, flow/session behavior, and truthful loss accounting.

Capture-only throughput is not a claim for full-feature stateful/NAT/HA operation.

Node performance is not cluster performance. HA qualification measures failover stages and capture/session/journal outcomes separately.

Hunter capacity is not sustainable ingest. Qualify ingest, verify/hash, indexes, query, retrieval, reprocessing, retention, scrub/resilver, and degraded-storage behavior separately.

ECC remains required direction for Hunter and a strong FW production preference until exact profiles are frozen.

## Net-Hunter Rules

Host owns FreeBSD, hardware, HBA/raw disks, ZFS, PF/networking, jail lifecycle, updates, and hardware/storage health.

Application jails do not receive host/root storage authority merely because they consume datasets.

Defined jails:

1. PCAP Data Ingest Jail
2. Record Processing Jail
3. External User Interface Jail
4. FW Configuration Backup Jail

External UI authoritative access is read-only. Writable UI state is non-authoritative workspace only.

Derived indexes may be rebuilt/reprocessed but MUST NOT rewrite authoritative PCAP/source journals.

### Net-Hunter records/search authority

`docs/NET-HUNTER-RECORDS.md` defines the governing record/search/reprocessing architecture.

Preserve two distinct catalog concerns:

```text
SEGMENT CATALOG
    where are the packets?

TRAFFIC / RECORD CATALOG
    what happened?
```

Do not make row-per-packet database storage a mandatory primary search model without measurement and explicit approval.

Every material traffic-derived fact must preserve source provenance sufficient to identify its authoritative source appliance/history object and processing/decoder lineage.

Historical MAC/IP/name/VLAN/control-plane relationships are time-bounded facts. Do not overwrite old relationships with current values.

Direct observation and correlated association must remain distinguishable.

Unknown, unsupported, malformed, partially parsed, and decoder-error traffic must remain discoverable where lower-level observation metadata permits.

### Search completeness

> **Stronghold must never present an incomplete index as complete history.**

A query over incomplete/rebuilding derived state must expose coverage or incompleteness rather than returning an unqualified `no results` conclusion.

Preserve distinctions such as:

```text
not observed
not captured
captured but not processed
processed but not decoded
decoded but not indexed
indexed and no result
history destroyed
history unavailable
```

When indexes are incomplete/unavailable, retain the ability to fall back toward the segment catalog and candidate authoritative PCAP rather than treating the system as historically blind.

### Reprocessing

Reprocessing creates new/superseding derived interpretation and MUST NOT alter source PCAP/journals or imply that later understanding was known at original observation time.

Preserve decoder/correlator identity, processing time/generation, source object lineage, and supersession state where applicable.

For significant reprocessing/schema changes, prefer building/validating a new derived generation before switching the default query generation rather than leaving a partially migrated live search state.

Targeted and full reprocessing operations are Hunter Processing Journal events.

### Processing boundaries and backlog

Record Processing Jail may read authoritative PCAP/source journals and write derived records/indexes, but MUST NOT rewrite authoritative history.

Distinguish:

```text
TRANSFER BACKLOG
INGEST BACKLOG
PROCESSING BACKLOG
REPROCESSING BACKLOG
INDEX REBUILD BACKLOG
```

Current receive/verify/durable commit work outranks historical reprocessing.

### PCAP pivot and export

Where source PCAP remains retained, traffic-derived results must preserve enough lineage to locate the applicable source segment(s).

A hunt-created PCAP export is a derived extract with export provenance; it is not the original authoritative segment even if its packet bytes are exact copies.

Historical decision correlation must use the configuration generation active at decision time, not today's policy merely because the Policy ID is unchanged.

## Resource Priority

FW priority intent:

```text
1. packet acquisition
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation/decision/journal append
5. HA heartbeat/control reserved lightweight capacity
6. journal checkpoint/finalization
7. local tier movement/backlog
8. HA bulk state sync
9. Hunter transfer
10. compression where approved
11. deeper analytics outside live path
```

Checkpoint signing, update work, bulk HA state, and analytics must throttle rather than hide capture loss.

## Explicit Deferrals

**VPN is deferred. IDS/IPS is deferred.**

Do not implement, select, or imply a current VPN architecture, IDS/IPS engine, inline IPS behavior, TLS interception, detection ruleset contract, or IDS/IPS fail-open/fail-closed behavior without later explicit approval.

Nothing built now should prevent future detection from reading authoritative PCAP and producing derived results, but detection is not a current capture dependency.

## Scope Discipline

Implementation begins with FW Phase 0 Traffic Observation Foundation.

Do not pull bridging/routing/enforcement, full journals, retention administration, HA, complete update manager, recovery/DR, encryption/key management, platform trust enforcement, complete Net-Hunter records/index/search/reprocessing, VPN, IDS/IPS, UI, Hunter processing, dynamic routing, VRF, or other future systems into Phase 0 without explicit approval.

Phase 0 may establish reproducible capture/hardware baselines useful to later qualification without implementing those later systems early.

Phase 0 segment/traffic catalogs must remain compatible with later Net-Hunter authority/provenance/search coverage requirements without implementing the complete later system.

## Repository Operations

Do not perform repository writes unless explicitly authorized for the specific action. This includes commit, push, merge, PR creation, branch/ruleset/settings changes, issues, deletion, and other writes.

Read-only inspection and local/offline work are permitted unless restricted. One approval does not carry forward.

## Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

Important failure paths should preserve enough state/history to answer what was observed, known, unknown, decided, actually performed, `NOT_PERFORMED`, failed, lost, and still recoverable.

For Hunter search, the same test includes whether a `no results` answer came from complete history/index coverage or merely from incomplete processing/decoding/indexing.

## Review Expectations

Before proposing a change as complete, verify that applicable capture, authority, routing/NAT/WAN, identity, time, journal, retention, HA, update, DR, crypto, platform-trust, hardware-qualification, Hunter-isolation, Net-Hunter provenance/search/reprocessing, and repository-write boundaries above remain intact.

## Nested AGENTS.md Files

Nested files may refine subtree requirements but must not silently weaken repository-wide capture, durability, integrity, truthfulness, authorization, identity/trust, time, journals, retention, HA, update, recovery, encryption, platform-trust, qualification, Net-Hunter isolation, Net-Hunter provenance/search/reprocessing, UI read-only, configuration-backup, or repository-write requirements.
