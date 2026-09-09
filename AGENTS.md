# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation work within the Stronghold repository. It is a behavioral contract and does not replace the governing architecture/roadmap documents:

- `docs/ARCHITECTURE.md`
- `docs/AF-XDP.md`
- `docs/NET-HUNTER-RECORDS.md`
- `docs/MANAGEMENT-PLANE.md`
- `docs/OBSERVABILITY.md`
- `docs/SUPPORT-DIAGNOSTICS.md`
- `docs/IDS-INSPECTION.md`
- `docs/SECURE-ACCESS.md`
- `docs/ROADMAP.md`

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **AF_XDP is the intended Stronghold FW packet-acquisition foundation.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **A secure tunnel is a protected path, not authorization to use every resource behind it.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

> **Record what the system can actually establish, preserve the source of that truth, and never manufacture certainty beyond it.**

> **An incomplete index is not complete history.**

> **Stronghold has one management authority and multiple management surfaces.**

> **Health is domain-specific; one healthy subsystem must not hide failure in another.**

> **Stronghold has no permanent vendor backdoor.**

> **Decrypted inspection payload is never intentionally persisted at rest on Stronghold FW.**

## Product Model

### Stronghold FW

Stronghold FW owns live-network responsibilities:

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
future selective TLS/application proxying only where explicitly configured
future zero-trust remote-access PEP / WireGuard termination
future L2TPv3/IPsec site-to-site / branch-office tunnel termination
```

### Stronghold Net-Hunter

Stronghold Net-Hunter owns historical responsibilities:

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
future isolated IDS/inspection analysis
future secure-access history/session correlation
```

Net-Hunter must never become a runtime dependency for FW capture, forwarding, NAT, enforcement, secure-access policy enforcement, or HA quorum/fencing. Future IDS/inspection processing also does not silently become a synchronous forwarding dependency.

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

Future decrypted inspection material is non-authoritative derived processing input. The authoritative packet history remains the original wire PCAP.

Secure-access session/tunnel correlation on Hunter is derived interpretation around authoritative source traffic and FW journals; it does not rewrite the original observation or decision history.

## Capture Rules

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline.

Unknown EtherTypes, unknown IP protocols, malformed/vendor-specific traffic, and Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

RAM is buffering only, not durable capture.

`docs/AF-XDP.md` defines the governing packet-acquisition architecture. **AF_XDP is the Phase 0 capture foundation.** AF_PACKET/TPACKET_V3 is not the planned primary Phase 0 implementation and must not be introduced as a silent fallback that preserves an AF_XDP qualification claim.

AF_XDP availability, native-driver operation, zero-copy capability, and Stronghold qualification are separate facts. Queue/RSS topology, UMEM/rings, worker affinity, CPU/NUMA/PCIe locality, NIC/driver/firmware behavior, offload representation, and drop accounting must be measured and exposed truthfully.

Keep the Phase 0 XDP program minimal and measurable. XDP/eBPF does not become a second policy/enforcement engine before the authoritative capture path is proven.

Live packet acquisition and durable PCAP writes outrank transfer, compression, indexing, analytics, support collection, debug work, future IDS/proxy work, future secure-access control/tunnel housekeeping, and other secondary activity.

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

Do not let route availability, NAT, FQDN association, WAN reachability, HA membership, future IDS availability, WireGuard peer state, L2TPv3 state, or IPsec SA state grant security permission.

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

A future qualified high-throughput profile may add a dedicated `INSPECTION` interface, but no such physical-role requirement is frozen now.

Routing is longest-prefix first, then equal-prefix source preference `STATIC`, `CONNECTED`, `DEFAULT`, `DYNAMIC`, then health, metric, deterministic tie-break.

NAT is separate from authorization. WAN eligibility is explicit per destination/service.

Future zero-trust remote access and site-to-site/branch-office tunneling must integrate with the same Stronghold networking/policy authority rather than creating an independent hidden routing/policy system.

## Common Truth Separations

Never collapse these distinctions:

```text
route available                    != route authorized
certificate valid                  != peer authorized
packet not decoded                 != packet not observed
no journal record                  != event did not occur
timestamp precision                != timestamp accuracy
policy allowed                     != forwarding succeeded
object expired                     != destruction authorized
bytes received                     != history durably committed
archived                           != destroyed
new interpretation                 != new historical observation
forwarding HA succeeded            != capture continuity guaranteed
virtual cluster identity           != physical capture identity
software installed                 != Stronghold update validated
configuration restored             != appliance validated
backup written                     != backup proven restorable
ZFS redundancy                     != disaster recovery
disk encrypted                     != secrets safely designed
TPM protected                      != disaster recoverable
cluster member                     != shared local disk key
key rotated                        != past exposure undone
entry written                      != journal entry durably committed
hash chain valid                   != checkpoint externally anchored
signed checkpoint                  != physically immutable storage
new journal epoch                  != uninterrupted history
system booted                      != approved system state
Secure Boot enabled                != measured boot verified
measured boot available            != attestation performed
Appliance ID                       != TPM identity
hardware detected                  != hardware qualified
hardware compatible                != hardware supported
link speed                         != validated dataplane rate
capture throughput                 != full-feature firewall throughput
node performance                   != HA cluster performance
storage capacity                   != sustainable ingest capacity
AF_XDP available                   != AF_XDP zero-copy available
AF_XDP zero-copy available         != Stronghold zero-copy qualified
queue configured                   != queue serviced adequately
worker running                     != AF_XDP ring healthy
database record                    != authoritative packet
no indexed result                  != no traffic
index unavailable                  != history unavailable
source history available           != index complete
reprocessing result                != original knowledge
timeline                           != authoritative journal
exported PCAP                      != original PCAP segment
current hostname association       != historical hostname association
direct observation                 != correlated association
processing failed                  != source history lost
native OS change                   != Stronghold configuration commit
configured state                   != operational state
operational state                  != historical state
API access                         != commit permission
same identity provider             != same active session
Hunter UI access                   != FW administration authority
diagnostic action                  != repair action
secret configured                  != secret readable
process running                    != subsystem healthy
interface UP                       != capture healthy
capture worker running             != PCAP durable
alert acknowledged                 != fault resolved
alert suppressed                   != event not recorded
alert resolved                     != historical gap erased
syslog delivered                   != journal committed
support access                     != configuration authority
support access                     != history-destruction authority
support bundle created             != support bundle transmitted
support PCAP                       != authoritative source segment
engineering shell exited           != platform state validated
Stronghold administration          != BMC administration
vendor-supported hardware          != Stronghold-qualified hardware
vendor-supported firmware          != Stronghold-qualified firmware
BMC access                         != Stronghold configuration authority
hardware change completed          != Stronghold platform validation passed
HSTS enabled                       != enterprise TLS inspection impossible
TLS proxied                        != TLS downgraded
client inspection cert valid       != origin certificate valid
inspection bypassed                != traffic unobserved
inspection feed lost               != authoritative capture lost
decrypted in FW memory             != decrypted retained on FW
HISTORY transport identity         != INSPECTION transport identity
encrypted inspection transport     != authorized inspection peer
IDS jail access                    != ZFS key-management authority
IDS finding retained               != full decrypted session retained
original encrypted PCAP            != decrypted inspection history
encrypted historical PCAP          != historical plaintext available
IDS storage unavailable            != permission to spool plaintext
IDS unavailable                    != production forwarding unavailable
deep detection                     != inline enforcement
inspection configured              != inspection coverage complete
WireGuard peer authenticated       != user authenticated
user authenticated                 != device authorized
device authorized                  != resource authorized
WireGuard tunnel established       != resource authorized
resource authorized                != connection succeeded
AllowedIPs configured              != Stronghold resource authorization
Access Session created             != application connection succeeded
REVOKE issued                      != revocation fully enforced
L2TPv3 tunnel established          != site authorized
IPsec SA established               != traffic authorized
remote network reachable           != remote network permitted
site tunnel healthy                != application connection succeeded
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
configuration_drift
inspection_bypassed
inspection_unsupported
inspection_feed_gap
access_session_expired
access_session_revoked
resource_not_authorized
site_tunnel_down
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

Journal-signing credentials are purpose-separated from TLS/history/HA, future IDS/inspection transport credentials, WireGuard peer credentials, and future site-tunnel credentials.

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

Future IDS findings and any retained decrypted context have retention independent from authoritative PCAP. Expiration of IDS material never grants authority to destroy authoritative PCAP.

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

Future secure-access HA must separately validate WireGuard/Access Session behavior and L2TPv3/IPsec tunnel failover/rekey state; ordinary forwarding failover does not prove secure-access continuity.

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

Stronghold uses purpose-separated encryption domains for packet history, journals/config history, secrets/private credentials, derived/temp data, and future IDS inspection persistence.

FW direction favors encrypted block devices beneath Btrfs/XFS so application capture remains simple and capture-first.

Hunter direction favors ZFS-native encrypted datasets aligned with host/jail authority.

TPM/HSM may protect normal key use but must not be the sole recovery path for authoritative history.

HA nodes do not share local disk keys merely because they share cluster state.

Private appliance identity keys should be non-exportable where supported. Replacement hardware normally generates new private credentials.

General configuration should reference secret IDs rather than repeat plaintext secrets.

Recovery authority is separate from normal administration and hunting.

Routine key rotation should not inherently require rewriting all historical PCAP. Key wrapping and full data re-encryption are distinct.

Key rotation does not prove previously protected data was never exposed.

Future IDS persistence uses a dedicated encrypted-at-rest inspection domain. The Hunter host owns ZFS/key-management authority; the IDS jail does not.

Future WireGuard and L2TPv3/IPsec credentials/keys remain purpose-separated from appliance identity, journal signing, HISTORY, HA, and IDS/inspection credentials.

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

Qualification binds release, CPU, NIC/driver/firmware, PCIe/NUMA topology, RAM/ECC, storage, platform trust, enabled feature set, and XDP/AF_XDP operating mode.

NIC qualification must cover visibility-affecting behavior: RSS/multi-queue, AF_XDP queue binding, UMEM/ring behavior, copy/zero-copy mode, VLAN/offload representation, filtering/promiscuous behavior, ring/drop behavior, MTU, timestamps, reset behavior, and firmware.

Performance testing must include bandwidth **and** PPS, multiple packet sizes, sustained duration, traffic mix, flow/session behavior, 64-byte packet stress, and truthful loss accounting.

Capture-only throughput is not a claim for full-feature stateful/NAT/HA, future TLS-inspection, or future secure-access operation.

Node performance is not cluster performance. HA qualification measures failover stages and capture/session/journal outcomes separately.

Hunter capacity is not sustainable ingest. Qualify ingest, verify/hash, indexes, query, retrieval, reprocessing, retention, scrub/resilver, degraded-storage behavior, and future IDS processing separately.

ECC remains required direction for Hunter and a strong FW production preference until exact profiles are frozen.

## Net-Hunter Rules

Host owns FreeBSD, hardware, HBA/raw disks, ZFS, PF/networking, jail lifecycle, updates, and hardware/storage health.

Application jails do not receive host/root storage authority merely because they consume datasets.

Defined current jails:

1. PCAP Data Ingest Jail
2. Record Processing Jail
3. External User Interface Jail
4. FW Configuration Backup Jail

Future optional jail:

5. IDS / Inspection Jail

External UI authoritative access is read-only. Writable UI state is non-authoritative workspace only.

Derived indexes may be rebuilt/reprocessed but MUST NOT rewrite authoritative PCAP/source journals.

The future IDS/Inspection Jail is isolated from authoritative PCAP write authority, FW configuration authority, host/root authority, retention/hold authority, and ZFS key-management authority.

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

> **Stronghold must never present an incomplete index as complete history.**

A query over incomplete/rebuilding derived state must expose coverage/incompleteness rather than returning an unqualified `no results` conclusion.

Reprocessing creates new/superseding derived interpretation and MUST NOT alter source PCAP/journals or imply that later understanding was known at original observation time.

Record Processing Jail may read authoritative PCAP/source journals and write derived records/indexes, but MUST NOT rewrite authoritative history.

Distinguish transfer, ingest, processing, reprocessing, and index-rebuild backlog.

Where source PCAP remains retained, traffic-derived results preserve enough lineage to locate applicable source segments.

## Management Plane Rules

`docs/MANAGEMENT-PLANE.md` defines the governing management-plane architecture.

CLI, API, and any future FW management UI are clients of one Stronghold management authority. No surface gets a separate configuration truth or bypass around candidate validation, authorization, commit, generation, or journaling.

Preserve:

```text
CONFIGURATION STATE
OPERATIONAL STATE
HISTORICAL STATE
```

Native Linux/FreeBSD state is implementation state below Stronghold. Direct native changes are not silently imported into Stronghold configuration; material divergence becomes drift/mismatch requiring explicit reconciliation/validation.

Local console/break-glass recovery and normal remote management are separate trust paths.

The Net-Hunter External UI remains an investigation surface and does not gain FW configuration authority.

Automation uses stable object/service identity rather than relying only on mutable names or shared human credentials.

Secret configuration is not ordinarily readable back in plaintext.

Preserve diagnose/test/repair behavioral distinctions.

Future WireGuard, L2TPv3, and IPsec native state remain implementation state below Stronghold and must not become a second unmanaged configuration authority.

## Observability / Alerting Rules

`docs/OBSERVABILITY.md` defines the governing observability architecture.

Health is domain-specific. Keep capture, PCAP durability, forwarding, policy, routing, WAN, HA, Hunter/history transfer, journals, time, trust, storage, hardware, and platform/update state independently observable.

Future IDS/inspection health is separate from capture/forwarding health. Inspection coverage, feed gaps, proxy state, IDS jail state, and inspection storage state must not be hidden by otherwise healthy forwarding.

Future secure-access health is also separate: PE/PA/PEP, identity/MFA/posture dependencies, WireGuard session state, Access Session lifecycle, L2TPv3 state, IPsec SA/rekey state, and site-tunnel health must not be collapsed into one `VPN_UP` bit.

Do not infer subsystem health from process uptime or interface-link state.

High-frequency metrics and authoritative journals are separate stores/purposes. Metrics measure; journals preserve meaningful authoritative transitions/actions.

Alert acknowledgement/suppression/resolution never rewrites journal history or erases a capture/durability/inspection gap.

External syslog, SNMP, API, and future webhook systems consume Stronghold state; they do not become authority for it.

Use secure SNMPv3 direction for new monitoring integration rather than designing around community-string SNMPv1/v2c.

No automatic packet/configuration/support telemetry is sent to Iron Signal Systems.

## Support / Diagnostics / Vendor OOB Rules

`docs/SUPPORT-DIAGNOSTICS.md` defines the governing support architecture.

Stronghold has no permanent vendor support account, hidden SSH key, hidden VPN, persistent reverse shell, or automatic remote-support tunnel.

Prefer structured normal diagnostics before native engineering access.

Support bundles are manifest-driven and sensitivity-aware. PCAP, secrets, private keys, and crash/core/memory content are never silently included.

Creating a support bundle is distinct from transmitting it. Support artifacts/telemetry are not automatically sent to Iron Signal Systems.

Future remote support, if implemented, is customer initiated, time bounded, scoped, visible, attributable, and revocable.

Support access does not imply configuration, retention/destruction, key-recovery, or HA administrative authority.

Native engineering/root access is exceptional. After native modification, perform drift/platform-qualification validation; do not silently treat native state as a Stronghold commit.

Support/debug/compression/diagnostic activity throttles before live capture is sacrificed.

Future decrypted inspection payload must never be silently written into debug logs, support bundles, core dumps, or plaintext temporary files. IDS/inspection crash material is high-sensitivity content.

### Vendor hardware management preference

When access below the Stronghold appliance layer is required, use the qualified hardware vendor's preferred/supported out-of-band management platform and procedures where practical.

Stronghold does not replace the vendor BMC/OOB platform. Stronghold validates the resulting hardware/firmware state against the Stronghold-qualified appliance profile.

BMC/OOB administration and Stronghold administration are separate trust domains.

Vendor-supported hardware/firmware does not automatically equal Stronghold-qualified hardware/firmware.

Vendor OOB access must not become a hidden path over PRODUCTION, HISTORY, or HA.

## Deferred Secure Access Rules

`docs/SECURE-ACCESS.md` defines the governing future secure-access architecture.

**Implementation remains deferred.** Do not implement a production Stronghold Access Agent, complete zero-trust PE/PA/PEP subsystem, WireGuard remote-access lifecycle, posture engine, L2TPv3/IPsec site tunnel, or secure-access HA/failover behavior without later explicit approval and phase placement.

The following architecture boundaries are already mandatory for any future implementation:

- remote/user access follows the NIST SP 800-207 Policy Engine / Policy Administrator / Policy Enforcement Point control-model direction;
- WireGuard is the core secure remote-access transport/data plane, not the Stronghold authorization database;
- WireGuard peer authentication and `AllowedIPs` never substitute for user/device/resource authorization;
- Stronghold Access Session state binds the transport peer to user, device, authentication/MFA/posture, policy generation, resource, service, duration, and authorization state as applicable;
- GRANT, DENY, and REVOKE are first-class control-plane outcomes;
- the Policy Administrator creates/removes authorized access-session and transport state according to Policy Engine decisions;
- Stronghold FW acts as the Policy Enforcement Point and applies normal Stronghold firewall authorization to tunneled traffic;
- resource-scoped authorization is preferred over granting broad internal network access merely because a tunnel is established;
- site-to-site / branch-office tunneling is a separate architecture from zero-trust remote access;
- site-to-site / branch-office direction uses L2TPv3 protected by IPsec;
- L2TPv3 tunnel state, IPsec SA state, remote reachability, and bridge/route availability never independently grant security permission;
- site/tunnel identity remains separate from current endpoint addressing and cryptographic session state;
- native WireGuard/L2TPv3/IPsec configuration does not become a second Stronghold configuration authority;
- secure-access operations remain observable, attributable, and journaled;
- Hunter correlation is optional derived analysis and does not become a runtime secure-access dependency;
- secure-access implementation remains subordinate to capture-first resource and qualification rules.

## Deferred IDS / TLS Inspection Rules

`docs/IDS-INSPECTION.md` defines the governing future inspection architecture.

**Implementation remains deferred.** Do not implement an IDS engine, TLS proxy, interception CA, IPS enforcement mechanism, or inspection fail-open/fail-closed behavior without later explicit approval and phase placement.

The following architecture boundaries are already mandatory for any future implementation:

- TLS/application inspection is explicitly selected by policy rather than required for ordinary forwarding.
- inspected application sessions remain encrypted on both sides of the FW proxy;
- decrypted inspection payload on FW is volatile-only and MUST NOT be intentionally persisted at rest;
- plaintext is transported to Hunter only by re-encrypting it over a dedicated mutually authenticated inspection channel;
- inspection transport uses purpose-specific IDS/inspection credentials and explicit peer authorization;
- HISTORY and INSPECTION identities, queues, accounting, health, and failure semantics remain separate;
- there is no plaintext fallback if inspection transport trust/encryption fails;
- future IDS processing occurs in an isolated IDS/Inspection Jail;
- any persisted IDS findings/context reside only in a dedicated encrypted-at-rest inspection domain;
- ZFS/key-management authority remains with the Hunter host, not the IDS jail;
- findings-only is the default retention direction; full decrypted session retention and TLS session-secret retention are not default behavior;
- normal IDS overload/Hunter unavailability does not silently backpressure production forwarding;
- any fail-closed/MUST_INSPECT behavior must be explicit and scoped;
- inspection bypass/unsupported states remain visible and do not imply the original traffic was unobserved;
- mTLS and pinned applications default toward bypass unless explicitly supported by a later approved proxy model;
- QUIC/HTTP3 behavior is explicit and never silently represented as successful native inspection;
- IDS findings remain derived interpretation with provenance toward authoritative encrypted-wire PCAP where available.

Deep Hunter analysis is not a per-packet synchronous forwarding oracle.

## Resource Priority

FW priority intent:

```text
1. AF_XDP packet acquisition / ring service
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation/decision/journal append
5. HA heartbeat/control reserved lightweight capacity
6. journal checkpoint/finalization
7. local tier movement/backlog
8. HA bulk state sync
9. authoritative Hunter HISTORY transfer
10. compression where approved
11. future transient IDS inspection transport / secure-access control housekeeping / deep analytics / observability / support work outside live path
```

Exact future ordering among secondary work must be qualified by measurement. Transient IDS work and secure-access control-plane/background work never outrank authoritative capture/history merely because those features are enabled.

Checkpoint signing, update work, bulk HA state, observability collection, support bundles, debug tracing, future IDS processing, secure-access housekeeping, and analytics must throttle rather than hide capture loss.

## Explicit Deferrals

**Secure-access implementation is deferred. IDS/IPS implementation is deferred.**

The future secure-access architecture is defined in `docs/SECURE-ACCESS.md`, but no production Access Agent, complete PE/PA/PEP implementation, WireGuard lifecycle, posture engine, L2TPv3/IPsec implementation, interoperability profile, or complete secure-access HA/failover contract is approved for implementation now.

The future inspection architecture is defined in `docs/IDS-INSPECTION.md`, but no IDS/IPS engine, detection ruleset, complete proxy implementation, QUIC implementation, complete fail-open/fail-closed contract, or IPS enforcement model is approved for implementation now.

Nothing built now should weaken authoritative PCAP capture or make secure access or detection a current Phase 0 dependency.

## Scope Discipline

Implementation begins with FW Phase 0 Traffic Observation Foundation using AF_XDP as the packet-acquisition foundation.

Do not pull bridging/routing/enforcement, full journals, retention administration, HA, complete update manager, recovery/DR, encryption/key management, platform trust enforcement, complete Net-Hunter records/index/search/reprocessing, complete management plane, complete observability/alerting system, support/remote-engineering system, secure-access/VPN implementation, IDS/IPS, TLS/application proxying, UI, Hunter processing, dynamic routing, VRF, or other future systems into Phase 0 without explicit approval.

Phase 0 may establish reproducible AF_XDP capture/hardware baselines and basic measurements required to prove capture correctness without implementing later management/observability/support/secure-access/IDS systems early.

Phase 0 segment/traffic catalogs must remain compatible with later Net-Hunter authority/provenance/search coverage requirements without implementing the complete later system.

## Repository Operations

Do not perform repository writes unless explicitly authorized for the specific action. This includes commit, push, merge, PR creation, branch/ruleset/settings changes, issues, deletion, and other writes.

Read-only inspection and local/offline work are permitted unless restricted. One approval does not carry forward.

## Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

Important failure paths should preserve enough state/history to answer what was observed, known, unknown, decided, actually performed, `NOT_PERFORMED`, failed, lost, and still recoverable.

For AF_XDP capture, the same test includes the active XDP/AF_XDP mode, queue/socket/worker state, ring pressure/starvation, relevant drop counters, and whether an observation gap exists.

For Hunter search, the same test includes whether a `no results` answer came from complete history/index coverage or merely incomplete processing/decoding/indexing.

For management/health/support work, the same test includes whether the operator can distinguish intended configuration from runtime state, determine which responsibility is degraded, and identify what diagnostic/support action changed or did not change appliance state.

For future secure access, the same test includes who/what requested access, which user/device/session/tunnel identity was established, what resource/site was authorized, which policy generation decided it, whether the secure path was actually established, whether access was granted/denied/revoked, whether revocation completed, and what traffic was actually permitted or failed.

For future IDS/inspection, the same test includes whether traffic was selected, successfully proxied, bypassed/unsupported, delivered to the IDS jail, dropped from the inspection feed, processed, and retained only according to the configured encrypted persistence policy.

## Review Expectations

Before proposing a change as complete, verify that applicable AF_XDP/capture, authority, routing/NAT/WAN, identity, time, journal, retention, HA, update, DR, crypto, platform-trust, hardware-qualification, Hunter-isolation, Net-Hunter provenance/search/reprocessing, management-plane, observability, support/vendor-OOB, future secure-access, future IDS/inspection, and repository-write boundaries remain intact.

## Nested AGENTS.md Files

Nested files may refine subtree requirements but must not silently weaken repository-wide AF_XDP/capture, durability, integrity, truthfulness, authorization, identity/trust, time, journals, retention, HA, update, recovery, encryption, platform-trust, qualification, Net-Hunter isolation, provenance/search/reprocessing, management-plane, observability, support/vendor-OOB, future secure-access, future IDS/inspection, UI read-only, configuration-backup, or repository-write requirements.