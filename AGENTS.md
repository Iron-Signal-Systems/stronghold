# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation should work within the Stronghold repository.

It is an engineering map and behavioral contract. It does not replace architecture and implementation contracts under `docs/`.

Stronghold is built around these governing principles:

> **Capture first. Never sacrifice observation for secondary work.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

Stronghold engineering must preserve truthful packet observation, explicit packet-loss accounting, durable capture semantics, authorization-first forwarding behavior, clear failure behavior, operational simplicity, identity/trust boundaries, timestamp truthfulness, journal lineage, retention/hold/destruction authority, HA ownership/fencing truthfulness, and the separation between Stronghold FW and Stronghold Net-Hunter.

## Product Model

### Stronghold FW

Stronghold FW runs on Arch Linux and owns live-network responsibilities:

```text
observe
record
authorize
bridge and/or route
enforce
journal decisions/state
maintain local history backlog
optionally participate in active/standby HA
transfer verified history to Net-Hunter
```

### Stronghold Net-Hunter

Stronghold Net-Hunter runs on FreeBSD with ZFS and jails and owns historical responsibilities:

```text
receive
verify
preserve
journal processing state
process
correlate
hunt
query
export
preserve FW configuration backups
manage authorized retention/hold lifecycle
```

Net-Hunter must never become a runtime dependency for Stronghold FW capture, bridging, routing, NAT, firewall enforcement, or HA quorum/fencing.

## Product Engineering Principles

### Capture first

Prefer work that improves, in order:

1. Wireshark-class visibility on configured physical interfaces;
2. packet capture correctness;
3. truthful loss accounting;
4. durable local capture;
5. integrity;
6. comprehensive source history/journals;
7. authorization/enforcement correctness;
8. explainability;
9. failure behavior;
10. operational simplicity; and
11. downstream processing.

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline.

Unknown EtherTypes, unknown IP protocols, malformed traffic, vendor-specific frames, and non-routable Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

Live packet capture and active durable PCAP writes take priority over transfer, compression, indexing, analytics, hunt activity, and other background work.

### Raw packet history is authoritative

Raw PCAPNG is authoritative for what network traffic Stronghold observed.

Structured observation, authorization, flow, protocol, bridge, firewall, routing, NAT, WAN-selection, HA, system, configuration, trust, time, retention, and Hunter-processing history exists to explain, correlate, locate, and make authoritative packet history usable.

Absence of a decoded/derived record is not proof traffic did not exist.

### Observation and disposition are different facts

Do not collapse physical observation, authorization, bridge forwarding, route selection, firewall action, NAT, WAN selection, HA ownership, egress interface, and egress VLAN into one ambiguous event.

Router-on-a-stick traffic may enter and leave the same physical interface under different VLAN contexts.

A DROP/REJECT does not suppress authoritative ingress capture.

Virtual HA network identity is forwarding identity, not capture identity. Historical provenance names the physical Stronghold node/interface that actually observed traffic.

### Authorization before normal forwarding work

```text
observe / capture
    ↓
early authorization
    ├── explicit deny → drop
    ├── no explicit allow → default drop
    └── authorized to proceed
             ↓
          routing
             ↓
      NAT/state/deeper work
             ↓
          final egress
```

Do not let a route, NAT rule, FQDN association, WAN path, or HA cluster membership grant security permission.

For Layer-7 policy, `authorized_to_continue_processing` is not equivalent to `final_allow`.

### Report only what Stronghold can establish

Preserve meaningful distinctions such as:

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
```

Do not infer success from absence of error. Do not report durability merely because data reached RAM or a write call returned.

Packet drops, capture gaps, journal/index lag, Hunter backlog, storage pressure, route failures, WAN degradation, HA sync/fencing failures, clock uncertainty, authentication failures, configuration failures, retention/destruction failures, and verification failures must remain observable.

### Journal, do not casually log

Maintain logically separate append-oriented journals for:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Committed entries are not silently overwritten. Corrections, recovery, retries, later success, retention/destruction, and superseding interpretation create new entries.

Preserve local advancing sequence/order independently of wall-clock correctness.

Critical journal integrity/tamper detection must remain feasible. Do not conflate journal-chain integrity with PCAP segment integrity.

### Retention is explicit policy

PCAP and each journal domain have independent retention policy.

Age/threshold establishes expiration eligibility, not destruction authority. Holds override normal expiration. Destructive retention must not rely on speculative parser-derived classification without an explicit approved contract.

### HA safety before aggressive failover

> **Loss of peer communication is not, by itself, proof that the peer is dead.**

> **A standby must not claim ACTIVE cluster identity until the fencing/election contract permits promotion.**

Split-brain prevention takes precedence over aggressive failover when ownership cannot be established safely.

Forwarding availability, session continuity, capture continuity, and historical durability are separate outcomes and must not be collapsed into one generic HA-success state.

### Simple does not mean vague

Do not hide Linux, FreeBSD, NIC, capture, bridge, routing, firewall, nftables/netfilter, filesystem, ZFS, jail, identity, PKI, clock, journal, retention, HA, storage, or failure behavior behind vague abstractions.

### Prefer explicit engineering

Prefer narrow, inspectable implementation over speculative frameworks. Do not create generic plugin systems, arbitrary execution frameworks, catch-all registries, or unnecessary service boundaries without a real requirement.

## Sources of Truth

`docs/ROADMAP.md` defines current phase scope and gates.

`docs/ARCHITECTURE.md` defines the broader Stronghold FW, Net-Hunter, capture, networking, policy, routing, HA, storage, history-transfer, identity, trust, authentication, time, journal, retention, jail, configuration, and read/write-boundary architecture.

Do not implement future systems merely because they are architecturally documented.

## Scope Discipline

Stronghold begins implementation with the FW capture foundation.

Do not pull Layer-2 bridging, routing, nftables enforcement, FQDN policy, NAT, adaptive WAN selection, remote authentication, PKI enrollment, journal implementation, full retention/hold administration, HA, VPN, IDS/IPS, external UI implementation, Net-Hunter processing, dynamic routing, VRF, or other future systems into the current phase unless explicitly approved.

## Contract Discipline

Code MUST NOT silently violate architecture/implementation contracts, and a contract MUST NOT be weakened merely to make code pass.

When implementation conflicts with a contract:

1. stop at the boundary;
2. identify the exact conflict;
3. determine whether implementation, contract, or requirement is wrong;
4. surface it explicitly; and
5. resolve intentionally before continuing.

Do not silently weaken capture completeness, loss accounting, durability, journal separation/ordering, authorization-before-routing, policy ordering, default deny, interface/VLAN identity, forwarding traceability, route rules, WAN eligibility, HA ownership/fencing, identity/trust separation, LDAPS-only AD behavior, timestamp truthfulness, retention/hold/destruction controls, integrity verification, storage migration safety, Hunter transfer verification, UI read-only boundaries, configuration-backup isolation, or lineage/provenance.

## Stronghold FW Rules

### Capture

The authoritative capture point is the configured supported physical interface. Capture does not require protocol recognition, routability, bridge relevance, or firewall relevance.

RAM is buffering only and not durable capture storage.

Initial acquisition direction is AF_PACKET/TPACKET_V3 unless measurement or later approved contract requires another source.

### FW networking model

Architecturally supported models:

```text
Layer-2 transparent bridging
Layer-3 IPv4/IPv6 routing
router-on-a-stick over 802.1Q trunks
hybrid Layer-2/Layer-3 deployments
```

Do not introduce a global forwarding mode that forces unrelated interfaces/VLANs into one behavior.

### FW objects and interface roles

VLAN is first-class and preserves both Stronghold identity and actual 802.1Q ID.

Expected objects include physical/logical interfaces, VLANs, bridge domains, zones, hosts, networks/groups, services/groups, FQDNs/groups, routes, security policy, NAT policy, WAN preference, HA cluster, HA nodes, and virtual forwarding identities.

Current interface roles:

```text
PRODUCTION
WAN
MANAGEMENT
HISTORY
HA
```

`MANAGEMENT`, `HISTORY`, and `HA` are not production transit paths.

`HISTORY` is not HA state sync. `HA` is not Hunter transfer. Neither is a normal interactive admin path.

Reject configurations that route production through HISTORY/HA, place MANAGEMENT/HA into production bridge domains, or otherwise violate role boundaries.

### FQDN truthfulness

DNS association is not proof a connection actually used a hostname. Preserve the source of Layer-7 hostname knowledge when later supported.

### Policy ordering

Stronghold is explicit-allow/default-deny.

Policy rules are dense integer positions, lowest to highest, first match wins. Policy ID is stable identity only and MUST NOT determine order.

### Routing

Only authorized traffic enters normal routing.

```text
1. longest prefix match
2. equal-prefix source preference:
       STATIC
       CONNECTED
       DEFAULT
       DYNAMIC
3. health / availability
4. metric
5. deterministic tie-break
```

A more-specific prefix wins. VRF is future-compatible, not initial.

### WAN preference

WAN eligibility is explicit per destination/service. Scheduled/adaptive selection may choose only among allowed WANs. Existing sessions normally remain on established WAN/NAT identity.

### NAT

NAT is separate from authorization. Preserve original and translated tuples and selected route/WAN/state.

### FW storage

```text
separate SSD / Btrfs direction for OS
NVMe / XFS HOT PCAP
local SSD / XFS WARM/backlog PCAP
Net-Hunter long-term history
```

PCAP exhaustion must not silently exhaust root filesystem.

### Net-Hunter independence

Hunter unavailability does not stop FW capture, bridge/routing, NAT, firewall enforcement, local journals, or HA election/forwarding. Hunter is not an HA witness/quorum dependency.

## Stronghold FW HA Rules

### Initial model

Initial HA architecture is optional **active/standby**. Do not silently convert it into active/active forwarding.

Each node keeps its own Appliance ID, physical NIC/MACs, management/history/HA identities, storage, clocks, source journals, certificates, and capture provenance.

The pair has a stable Cluster ID.

### Cluster forwarding identity

Cluster-owned virtual gateway/WAN IPs and virtual MACs may move between nodes. Only the ACTIVE node may claim them.

A STANDBY may hold configuration for virtual identity but MUST NOT answer/forward as owner until the fencing/election contract permits activation.

Virtual identity movement never changes historical physical capture provenance.

### Routed/VLAN and transparent HA

Routed/VLAN HA uses cluster-owned gateway identities. Static/transferable WAN identity may likewise be cluster-owned.

Transparent L2 HA uses explicit bridge ownership. Only one cluster bridge path may forward; the standby path must not create an L2 loop.

Dynamic-WAN HA such as DHCP/PPPoE requires a later protocol-specific contract.

### Cluster and node-local configuration

Keep cluster configuration distinct from node-local configuration.

Cluster configuration includes policy, VLANs/zones/bridge domains, routes, NAT, WAN preference, and virtual forwarding identity.

Node-local configuration includes Appliance ID, management/history/HA addressing, physical NIC/PCI mapping, local storage mapping, and node certificate identity.

### Cluster commit behavior

Normal cluster commits must validate cluster semantics and each node's hardware/local context, stage the generation appropriately, and expose synchronization state.

A stale standby is a degraded failover target.

If degraded commits are later permitted while a peer is unavailable, they require explicit privileged action and must journal/configure the mismatch truthfully.

Failover does not create a new configuration generation merely because ACTIVE ownership changed.

### State synchronization

Synchronize only state justified for stateful failover, such as:

```text
firewall session state
NAT mappings
session timers
selected WAN
route/session binding
virtual runtime identity
```

Maintain explicit sync sequence/health such as:

```text
IN_SYNC
MINOR_LAG
DEGRADED
OUT_OF_SYNC
UNKNOWN
```

Do not promise universal seamless session migration. Record continuity limitations truthfully.

### Heartbeat/control versus bulk state

Heartbeat/control and bulk state synchronization are logically separate traffic classes.

Heartbeat/control must remain lightweight and must not be starved by bulk sync.

Capture remains the highest live-data priority. Bulk HA sync may lag under pressure.

Do not treat one missed heartbeat as automatic failover authorization.

### Promotion and fencing

Standby promotion must evaluate local health, required path readiness, peer availability evidence, fencing/election safety, configuration state, and state-sync condition.

No single heartbeat timeout may bypass the fencing/election contract.

When peer ownership cannot be established safely, prefer a single-owner outage over two nodes simultaneously claiming the cluster forwarding identity.

A secondary management observation path may help distinguish an HA-link failure from peer failure, but lack of management reachability alone does not prove the peer is dead.

### HA trust

HA messages require authenticated explicit cluster membership. A device plugged into the HA network must not be able to claim ACTIVE, influence election, or inject trusted state merely because it can send packets.

Exact HA protocol/certificate/fencing implementation remains future work.

### Manual failover and maintenance

Support architecture for controlled manual failover and explicit `MAINTENANCE` state. Planned transitions must remain distinguishable from failure-driven transitions.

Expected runtime states may include:

```text
ACTIVE
STANDBY
PROMOTING
DEMOTING
MAINTENANCE
DEGRADED
FAILED
```

### HA is not PCAP replication

Do not create a hidden requirement that every packet must be mirrored to the standby before capture is considered successful.

The ACTIVE node captures locally and transfers to Hunter normally. The standby synchronizes operational state, not a mandatory realtime duplicate PCAP stream.

If a failed node contains untransferred PCAP, that history remains attributable to that node and must be reconciled/recovered according to the later segment recovery contract.

Successful forwarding failover does not justify a claim of zero packet-history gap.

### HA journaling

HA runtime events belong in the System/Health Journal; human-triggered operations also belong in the Administrative Journal.

Preserve facts such as Cluster ID, node Appliance ID, role transition, configuration generation, state-sync sequence/health, reason, fencing result, virtual identity assumption, and failover result.

## Administrative Authentication and Authorization Rules

### AD is LDAPS only

Active Directory authentication MUST use LDAPS only. No plaintext LDAP and no LDAPS→LDAP downgrade.

Validate configured LDAPS trust/server identity. Validation failure is authentication failure, not permission to weaken transport.

### External authentication

Remote authentication may support LDAPS-based AD, RADIUS, and TACACS+ according to later scope. Protected local identity remains for installation/recovery/break-glass and is not an invisible fallback.

### Authorization separation

Authentication is not authorization.

Keep viewing config, candidate edit, validate, commit, rollback, system/network/security administration, HA administration, hunting, PCAP export, hold administration, retention administration, and manual destruction separable.

Net-Hunter access is not FW administration authority.

### Management ingress

Normal remote administration belongs on MANAGEMENT. HISTORY and HA are not normal interactive admin paths.

### Administrative journal

Authentication attempts, MFA state where known, sessions, role changes/use, config operations, HA manual actions, exports, holds, retention/destruction, break-glass, update/reboot/shutdown, and other privileged activity belong in Administrative Journal.

Never journal passwords, TOTP secrets/codes, private keys, or plaintext credentials.

## Appliance Identity and Trust Rules

Each appliance has stable Appliance ID independent of hostname/IP/current certificate.

History transport uses mTLS and explicit peer authorization. Certificate validity establishes identity but not blanket authorization.

Certificate rotation preserves Appliance ID. Expiry/revocation/peer rejection blocks transfer and creates backlog/degraded state, not dataplane shutdown.

Management/UI and history-transfer certificates are separate purposes.

HA cluster membership likewise requires authenticated explicit membership.

## Time and Timestamp Rules

Store authoritative wall-clock timestamps in UTC. Timezone is presentation only.

Preserve journal/event ordering independently of wall-clock correction with sequence/monotonic context where appropriate.

Initial sync supports multiple NTP sources; NTS/PTP/hardware timestamping are later capabilities.

Preserve clock states such as synchronized, holdover, unsynchronized, and fault.

Do not equate timestamp resolution with accuracy.

Hunter preserves source FW time separately from Hunter receive/verify/process time.

## Journal Rules

Maintain separate journals:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Committed entries are append-oriented. Corrections, retries, recovery, retention/destruction, later success, and superseding interpretation create new entries.

Each journal must preserve stable entry identity, source Appliance ID, local order/sequence, timestamp, type/result, and relevant references such as configuration generation, policy/route/segment/peer/cluster identity.

Traffic Decision Journal explicitly preserves `NOT_PERFORMED` when later pipeline work did not occur.

Hunter preserves source FW journals and appends its own processing history without rewriting source history.

## Retention, Hold, and Destruction Rules

PCAP and each journal domain have independent retention policy.

Age/threshold is eligibility, not destruction authority. Holds override normal retention.

Manual destruction requires explicit privileged authority and reason. Destruction never silently removes the fact that the object existed; journal applicable identity/source/time/integrity/policy/reason/authority/result.

Archive movement is not destruction and preserves identity/integrity/lineage/retrieval state.

## Dedicated History Network Rules

Stronghold FW and Hunter use dedicated history transfer. Current physical direction includes 10/25/40 GbE deployment options.

History transfer is verified mTLS handoff, not blind copy/delete. Hunter MUST NOT ACK until required destination finalization, verification, and durable commit complete, even under storage pressure.

## Net-Hunter Host and Jail Rules

Net-Hunter runs FreeBSD/ZFS/jails. The host owns physical hardware, HBA/raw disks, ZFS, host networking/PF, jail lifecycle, updates, and hardware/storage health.

Application jails do not receive raw host/storage authority merely because they consume datasets.

### PCAP Data Ingest Jail

Receives finalized PCAP/source journals from explicitly authorized FW appliances, verifies and commits, then ACKs verified receipt. No external hunt access.

### Record Processing Jail

Reads authoritative PCAP/source history and produces/enriches searchable records/indexes. It may reprocess history but not rewrite authoritative source history.

### External UI Jail

Read-only for authoritative history. It may hunt/query/search/correlate/view/retrieve/export/report, using writable non-authoritative workspace only.

It must not modify authoritative PCAP/source journals, silently rewrite processed history, delete outside authorized retention workflow, change ingest state, modify FW config backups, or change FW policy through hunt UI.

### FW Configuration Backup Jail

Preserves versioned FW configuration backups. Normal access is authorized FW appliances and local Net-Hunter administration only.

## State and Failure Discipline

Failure behavior is part of the product. Test failure paths with success paths for capture, durability, destructive, privileged, auth/trust, clock, HA, and security-sensitive operations.

Examples include:

```text
packet drop
capture pressure
short write / fsync failure
storage full
power loss
segment interruption
hash mismatch
transfer failure
Hunter unavailable/full
authentication/LDAPS failure
peer certificate/authorization failure
clock fault
journal unavailable
ZFS/jail failure
hold/destruction failure
HA link failure
HA state-sync lag/out-of-sync
HA config mismatch
promotion blocked
fencing failure
split-brain detection
bridge/route/policy application failure
```

A failed operation is still factual history. A later retry does not erase it.

## Durability Rules

`write()` success is not durability. Do not report durable state until the required OS/filesystem durability boundary completes.

Interrupted artifacts should be preserved where needed for truthful reconciliation.

## Storage, Pressure, and Migration Rules

Only finalized closed segments may be migrated/transferred. A source capture is not deleted merely because destination write/transfer completed; destination finalization, integrity verification, durable commit, and ACK must complete according to contract.

FW storage pressure should expose NORMAL/HIGH/URGENT/CRITICAL. Pressure may throttle secondary work and accelerate safe transfer/tiering, but cannot silently create emergency deletion authority.

By default do not automatically delete unacknowledged authoritative FW history merely to hide a full disk. If exhaustion prevents durable capture, report the history gap.

Hunter pressure preserves ingest/verification/commit first, reduces nonessential processing, and runs only authorized retention not blocked by holds. If Hunter cannot commit, fail ACK so FW retains backlog.

## Secrets and Sensitive Material

Do not intentionally log/journal plaintext credentials, private keys, passphrases, TOTP secrets/codes, or other authentication secrets outside approved secure storage.

Packet captures and firewall configuration backups are highly sensitive. Debug/support/test/export/temp data must not accidentally contain real customer secrets/capture/configuration.

## Repository Operations

Do not perform repository writes unless explicitly authorized for the specific action.

This includes commit, push, merge, PR creation, branch/ruleset/settings changes, issue creation, deletion, and other writes.

Read-only inspection and local/offline working changes are permitted unless restricted. Permission for one write does not carry forward.

## Review Expectations

Before proposing a change as complete, verify at minimum:

- Wireshark-class capture visibility remains intact;
- capture priority and raw PCAP authority remain intact;
- packet-loss/degraded states remain truthful;
- physical observation stays distinct from policy/route/NAT/WAN/HA disposition;
- default deny and authorization-before-routing remain intact;
- Policy ID is not priority;
- route selection and explicit WAN eligibility remain intact;
- AD has no plaintext LDAP/downgrade;
- authentication/authorization/journaling remain separate;
- appliance identity is distinct from certificate/IP/hostname;
- timestamp resolution is not presented as accuracy;
- journals remain separate/append-oriented;
- retention eligibility is not destruction authority;
- holds override expiration;
- storage pressure cannot silently delete unacknowledged history;
- Hunter cannot ACK uncommitted history;
- HA virtual forwarding identity is not substituted for physical capture provenance;
- standby cannot claim ACTIVE merely because one heartbeat path failed;
- HA control cannot be starved by bulk state sync;
- configuration/state synchronization is explicit in promotion decisions;
- split-brain prevention/fencing remains explicit;
- Net-Hunter is not introduced as HA quorum/dataplane dependency;
- External UI authoritative access remains read-only;
- FW config backup isolation remains intact;
- no future phase was accidentally pulled forward; and
- applicable nested `AGENTS.md` checks were run/reported.

## Nested AGENTS.md Files

Nested files may refine requirements for their subtree but must not silently weaken repository-wide capture, durability, integrity, truthfulness, authorization, identity/trust, time, journal, retention, HA, Net-Hunter isolation, UI read-only, configuration-backup, or repository-write requirements.
