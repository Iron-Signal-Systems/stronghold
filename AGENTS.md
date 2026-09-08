# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation should work within the Stronghold repository.

It is an engineering map and behavioral contract. It does not replace architecture and implementation contracts under `docs/`.

Stronghold is built around these governing principles:

> **Capture first. Never sacrifice observation for secondary work.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

Stronghold engineering must preserve truthful packet observation, explicit packet-loss accounting, durable capture semantics, authorization-first forwarding behavior, clear failure behavior, operational simplicity, identity/trust boundaries, timestamp truthfulness, journal lineage, and the separation between Stronghold FW and Stronghold Net-Hunter.

## Product Model

Stronghold is a two-appliance system.

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
```

Net-Hunter must never become a runtime dependency for Stronghold FW capture, bridging, routing, NAT, or firewall enforcement.

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

The first capture requirement is explicit:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

A supported capture path should provide the same class of interface visibility expected from Wireshark/dumpcap operating on the same supported physical interface under the same conditions.

Unknown EtherTypes, unknown IP protocols, malformed traffic, vendor-specific frames, and non-routable Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

Live packet capture and active durable PCAP writes take priority over transfer, compression, indexing, analytics, hunt activity, and other background work.

Secondary work may throttle, pause, or fall behind. Stronghold must not deliberately sacrifice live capture merely to keep secondary systems current.

### Raw packet history is authoritative

Raw PCAPNG is authoritative for what network traffic Stronghold observed.

Structured observation, authorization, flow, protocol, bridge, firewall, routing, NAT, WAN-selection, system, configuration, trust, time, and Hunter-processing history exists to explain, correlate, locate, and make the authoritative packet history usable.

Failure to recognize, decode, enrich, index, or correlate traffic must not cause the underlying packet to be discarded.

Absence of a decoded/derived record is not proof that the corresponding traffic did not exist.

### Observation and disposition are different facts

Stronghold must preserve the distinction between what a physical interface observed and what the appliance later did with the traffic.

Do not collapse ingress observation, authorization, bridge forwarding, route selection, firewall action, NAT, WAN selection, egress interface, and egress VLAN into one ambiguous event.

Router-on-a-stick traffic may enter and leave the same physical interface under different VLAN contexts. Preserve the correlation without pretending the observations are unrelated traffic.

A DROP or REJECT disposition must not by itself suppress authoritative ingress capture.

### Authorization before normal forwarding work

A packet must not enter normal routing/NAT/deeper-forwarding work merely because a viable route exists.

Stronghold semantics are:

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

Reject clearly unauthorized traffic as early and cheaply as practical on the qualified dataplane without weakening capture visibility or correctness.

Do not let the existence of a route, NAT rule, FQDN association, or WAN path grant security permission.

For Layer-7 policy, `authorized_to_continue_processing` is not equivalent to `final_allow`. Do not report a final Layer-7 allow until the required Layer-7 fact has actually been established.

### Report only what Stronghold can establish

Stronghold must not convert incomplete observations into conclusions.

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

Do not infer a positive state from the absence of an error.

Do not report a packet as durably captured merely because it reached RAM or because a write call returned successfully.

Packet drops, capture gaps, journal/index lag, Net-Hunter transfer backlog, storage pressure, route failures, WAN degradation, clock uncertainty, authentication-service failures, configuration failures, and verification failures must remain observable.

### Journal, do not casually log

Do not build Stronghold authoritative operational history around generic mutable text logs.

Maintain distinct append-oriented journal domains for:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

These domains are logically separate even if future storage implementation shares lower-level infrastructure.

Committed entries are not silently edited or overwritten. Corrections, superseding state, recovery, retries, retention actions, and later success are represented by new entries.

Preserve local advancing sequence/order independently of wall-clock correctness.

Critical journal integrity/tamper detection should remain possible. Do not choose a storage model that makes later append-chain or equivalent verification impractical without a concrete reason.

PCAP integrity remains segment-oriented; do not introduce per-packet cryptographic chaining merely because journals are append-oriented.

### Simple does not mean vague

Stronghold should remain operationally simple while retaining authoritative technical detail.

Do not hide Linux, FreeBSD, NIC, packet-capture, bridge, routing, firewall, nftables/netfilter, filesystem, ZFS, jail, identity, PKI, clock, journal, storage, or failure behavior behind vague abstractions.

### Prefer explicit engineering

Prefer simple, narrow, inspectable implementation over speculative abstractions.

Do not create generic plugin frameworks, arbitrary execution systems, catch-all platform frameworks, generalized registries/factories, or unnecessary service boundaries for anticipated future needs.

Implement the requirement that exists now.

## Sources of Truth

### Phase scope and sequence

`docs/ROADMAP.md` defines current implementation phase scope, sequencing, targets, and exit gates.

The broader architecture may be documented before its implementation phase begins. Do not implement future systems merely because they are already described architecturally.

### Architecture invariants

`docs/ARCHITECTURE.md` defines the current Stronghold FW, Net-Hunter, capture, networking, policy, routing, storage, history-transfer, identity, trust, authentication, time, journal, jail, configuration, and read/write-boundary architecture.

Future dedicated contracts may refine implementation details without silently weakening these invariants.

## Scope Discipline

Work only within the current roadmap phase and current engineering slice unless explicitly directed otherwise.

Stronghold begins implementation with the FW capture foundation. Do not pull Layer-2 bridging, routing, nftables enforcement, FQDN policy, NAT, adaptive WAN selection, remote authentication, PKI enrollment, journal implementation, VPN, IDS/IPS, external UI implementation, Net-Hunter processing, HA, dynamic routing, VRF, or other future systems into the current phase unless explicitly approved.

Future compatibility may be preserved where useful, but future features should not be implemented early without a concrete current requirement.

## Contract Discipline

Code MUST NOT be changed to silently violate an architecture or implementation contract.

A contract MUST NOT be changed merely to make failing code pass.

When implementation conflicts with an existing contract:

1. stop at that boundary;
2. identify the exact conflict;
3. determine whether the implementation is wrong, the contract is wrong, or the requirement was misunderstood;
4. surface the issue explicitly; and
5. resolve the discrepancy intentionally before continuing through that boundary.

Do not silently weaken capture completeness claims, packet-loss accounting, durability requirements, journal separation/ordering, authorization-before-routing behavior, policy ordering, default-deny behavior, interface/VLAN identity, forwarding-disposition traceability, route-selection rules, WAN eligibility, identity/trust separation, LDAPS-only AD behavior, timestamp truthfulness, integrity verification, storage migration safety, Net-Hunter transfer verification, UI read-only boundaries, configuration-backup isolation, retention behavior, or lineage/provenance.

## Stronghold FW Rules

### Capture Requirement #1 — Wireshark-class interface visibility

The authoritative capture point is the configured supported physical interface.

If traffic is presented to that interface and observable through the supported NIC/driver capture path, Stronghold records it without first requiring protocol recognition, decoding, routability, bridge relevance, or firewall relevance.

This includes, when presented to the interface, ordinary IP traffic and Layer-2/control-plane traffic such as ARP, DHCP, DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes, unknown IP protocols, vendor-specific frames, and malformed traffic.

Stronghold must not claim to have observed traffic that the NIC, upstream topology, hardware filtering, or driver did not present to the supported capture path.

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline for capture validation.

RAM is buffering only and is not durable capture storage.

The initial preferred packet-acquisition direction is Linux AF_PACKET with TPACKET_V3 unless measurement or a later approved contract requires another source.

Any capture-source abstraction must be justified by a real implementation boundary rather than speculative portability.

### FW networking model

Stronghold FW may support different forwarding models on different configured portions of the same appliance.

Architecturally supported models include:

```text
Layer-2 transparent bridging
Layer-3 IPv4/IPv6 routing
router-on-a-stick over 802.1Q trunks
hybrid deployments combining Layer 2 and Layer 3 behavior
```

Do not introduce a global forwarding-mode abstraction that unnecessarily forces all interfaces/VLANs into the same mode.

Layer-2 forwarding belongs to explicit bridge-domain context. Layer-3 termination belongs to explicit logical-interface context.

Connected, static, and default routing are the initial intended routing direction. Dynamic routing remains future work unless explicitly approved.

### FW network objects and roles

VLAN is a first-class Stronghold object.

Do not represent VLAN meaning only as an anonymous integer buried in a Linux subinterface name.

Records and configuration must preserve both Stronghold object identity and the actual 802.1Q VLAN ID.

The architecture also anticipates explicit physical-interface, logical-interface, bridge-domain, zone, host, network, address-group, service, service-group, FQDN, FQDN-group, route, security-policy, NAT-policy, and WAN-preference-policy objects.

One routed logical interface belongs to one security zone. A zone may contain multiple logical interfaces/VLANs.

`MANAGEMENT` and `HISTORY` interface roles are not production transit paths. Reject configurations that turn them into normal production forwarding paths.

### FQDN truthfulness

DNS-backed FQDN policy and directly observed Layer-7 hostname identity are different facts.

Do not claim that a connection used a hostname merely because its destination IP appeared in a DNS resolution set.

When Layer-7 hostname information is later supported, preserve the source of that knowledge, such as TLS SNI, HTTP Host, DNS correlation, or `not_observed`.

### Policy ordering

Stronghold is explicit-allow/default-deny.

Policy rules are evaluated by dense current integer position, lowest to highest; first match wins.

A Policy ID is stable identity only and MUST NOT be used as priority/order.

Inserting/moving rules shifts the affected range and preserves a dense ordered list.

Historical decision history must preserve policy identity, position at decision time, action, and configuration generation.

### Routing

Only authorized traffic enters normal routing.

Route selection is:

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

A more-specific prefix wins regardless of route-source preference.

VRF is future-compatible, not an initial requirement. Preserve a routing-domain boundary without implementing multi-VRF behavior early.

### WAN preference

Stronghold does not assume alternate-WAN permission.

WAN eligibility is explicitly configured per destination/service policy.

Scheduled/adaptive selection may choose only among explicitly allowed WANs.

Adaptive path measurements may include reachability, RTT, loss, and jitter, with hysteresis. Existing sessions normally remain on their established WAN/NAT identity.

### NAT

NAT is separate from security authorization and never grants permission by itself.

Preserve original and translated tuples and the associated selected route/WAN/state.

### FW storage direction

The current preferred FW storage model is:

```text
separate SSD / Btrfs direction for Arch Linux OS
NVMe / XFS direction for HOT PCAP capture
local SSD / XFS direction for WARM/backlog PCAP
Net-Hunter for long-term history
```

PCAP storage must not share the root filesystem in a way that allows capture exhaustion to silently exhaust the operating system.

### Net-Hunter independence

Net-Hunter unavailability must not stop Stronghold FW capture, bridging, routing, NAT, firewall enforcement, or local source-journal creation while local FW resources remain capable of operating.

A Net-Hunter outage creates explicit transfer backlog/degraded state.

Do not silently discard pending history or claim successful transfer when Net-Hunter has not verified and acknowledged it.

## Administrative Authentication and Authorization Rules

### AD is LDAPS only

Active Directory authentication MUST use LDAPS only.

Do not implement plaintext LDAP authentication and do not automatically downgrade from LDAPS to LDAP when certificate validation or secure connectivity fails.

Validate the configured LDAPS trust/server identity requirements. A validation failure is a secure-authentication failure, not permission to weaken the transport.

### External authentication

Remote authentication may support LDAPS-based AD, RADIUS, and TACACS+ according to later implementation scope.

Protected local appliance identity remains available for installation/recovery/break-glass and must not be an invisible remote fallback.

External authentication failure must not affect the dataplane.

### Authorization separation

Authentication is not authorization.

Keep important privileges separable, including viewing config, editing candidate config, validating, committing, rollback, system administration, hunting, and PCAP export.

Viewing Hunter history is not equivalent to permission to export packet data.

Net-Hunter access is not FW administration authority.

### Management ingress

Normal remote administration belongs on the `MANAGEMENT` role/interface and may be source-restricted.

The `HISTORY` interface is not a normal interactive administration path.

### Administrative journal

Authentication attempts, MFA state where known, session lifecycle, role use/change, candidate/config actions, commits, rollbacks, exports, break-glass use, update/reboot/shutdown, and other privileged activity belong in the Administrative Journal.

Never log passwords, TOTP secrets/codes, private keys, or other plaintext credentials.

## Appliance Identity and Trust Rules

Each appliance has a stable Stronghold Appliance ID independent of hostname, IP address, or current certificate.

History transport uses mTLS and explicit peer authorization.

Certificate validity establishes cryptographic identity; it does not grant blanket permission to use a Hunter.

Joining/pairing a FW to a Hunter is explicit and revocable. Discovery alone does not establish trust.

Certificate rotation preserves Appliance ID.

Certificate expiry/revocation/peer rejection causes transfer failure/backlog/degraded state, not dataplane shutdown.

Management/UI certificates and history-transfer certificates are separate purposes.

TLS success does not replace segment identity/hash/integrity verification, destination commit, or acknowledgement.

The architecture favors a Stronghold-specific appliance trust hierarchy. Do not couple Stronghold history-link identity to AD PKI merely for convenience without an approved architecture change.

Trust enrollment, rotation, revocation, rejection, and relevant authentication-trust failures belong in the Trust / Identity Journal.

## Time and Timestamp Rules

Store authoritative wall-clock timestamps in UTC. Timezone is presentation only.

Preserve journal/event ordering independently of wall-clock corrections with an advancing sequence/ordering context and monotonic time where appropriate.

A backward wall-clock adjustment must not reverse journal order.

Initial synchronization should support multiple configured NTP sources. NTS/PTP/hardware timestamping are future capabilities where justified, not assumptions.

Preserve meaningful clock states such as synchronized, holdover, unsynchronized, and fault.

Clock/source transitions and significant adjustments belong in the Time / Clock Journal.

Do not equate timestamp resolution with timestamp accuracy.

Hunter preserves source FW time separately from Hunter receive/verify/process time.

Time synchronization failure must not stop capture/routing/firewall operation; it creates explicit degraded clock-confidence state.

## Journal Rules

### Domain separation

Maintain logically separate journals for:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Do not silently collapse them into one generic mutable logfile because implementation convenience makes that easy.

### Append-oriented behavior

Committed journal entries are append-only in semantic behavior.

Corrections, superseding interpretations, retries, recovery, later success, retention, and destructive actions create new entries.

Do not rewrite an old entry to make current state look as though the earlier state/failure never occurred.

### Ordering and identity

Each journal must preserve stable entry identity, source Appliance ID, local advancing order/sequence, origin timestamp, entry type/facts/result, and relevant references such as configuration generation, policy/route/segment/peer identity.

Exact schemas/UUID/hash contracts remain future work.

### Integrity

Journal storage/design must preserve the ability to implement cryptographic append/tamper verification.

Do not conflate journal-chain integrity with PCAP segment integrity.

### Traffic Decision Journal

Preserve what was done and what was deliberately not done.

For example, an early deny may truthfully record:

```text
authorization: DENY
routing: NOT_PERFORMED
WAN selection: NOT_PERFORMED
NAT: NOT_PERFORMED
final disposition: DROP
```

An allow with no route may record:

```text
authorization: ALLOW
routing: FAILED
reason: NO_ROUTE
NAT: NOT_PERFORMED
final disposition: DROP
```

Do not infer later pipeline work when it did not occur.

### Hunter processing

Net-Hunter preserves source FW journals and appends its own receive/verify/commit/process/reprocess/export/retention events.

Hunter does not rewrite the originating FW journal entry.

Derived analytical records may be superseded by new derived interpretation but must retain lineage to source PCAP/journal facts.

## Dedicated History Network Rules

Stronghold FW and Net-Hunter use a dedicated history-transfer path.

Current physical-link direction includes 10 GbE, 25 GbE, and 40 GbE interfaces as deployment options.

History-link speed is independent of the validated Stronghold FW dataplane rating. Do not infer or advertise a firewall forwarding/capture capability from the speed of the dedicated history interface.

History transfer is a verified mTLS-authenticated handoff, not a blind copy/delete operation.

A source segment/journal batch must remain distinguishable from destination receipt, finalization, verification, commit, and acknowledgement.

## Net-Hunter Host Rules

Stronghold Net-Hunter runs on FreeBSD with ZFS and jails.

The FreeBSD host owns infrastructure authority, including:

```text
physical hardware
HBA/raw disk visibility
ZFS pools and datasets
host networking
PF
jail lifecycle
FreeBSD updates
hardware/storage health
```

Do not give application jails raw host, HBA, disk, or ZFS-pool administration merely because they consume storage.

## Net-Hunter Jail Rules

The current application-jail architecture has four roles.

### PCAP Data Ingest Jail

The ingest jail receives finalized PCAP segments, source journals, and associated initial records from explicitly authorized Stronghold FW appliances over the approved history trust path.

It may perform peer authentication/authorization, transfer-completeness checks, integrity verification, commit work, and verified receipt acknowledgement.

It does not provide external hunt/query user access.

### Record Processing Jail

The record-processing jail reads authoritative PCAP and source FW history and produces/enriches searchable records and indexes.

It may reprocess historical PCAP when future decoders improve.

Normal processing must not rewrite authoritative packet or source-journal history.

### External User Interface Jail

The External UI jail is read-only with respect to authoritative Stronghold traffic/history data.

It may:

```text
hunt
query
search
correlate
build timelines
view records/journals
retrieve referenced PCAP
create controlled derived exports
create reports/notes
view status
```

It MUST NOT:

```text
modify authoritative PCAP
modify source journals
modify observation records
silently rewrite processed records/indexes
delete historical data outside an explicitly authorized retention/destructive workflow
change ingest state
change retention policy without separate authorization
modify FW configuration backups
modify Stronghold FW policy/configuration through the hunt interface
```

Writable UI storage is limited to non-authoritative material such as session state, temporary query work, derived PCAP exports, reports, notes, and download staging.

A derived export is not authoritative Stronghold history.

### FW Configuration Backup Jail

The FW Configuration Backup jail preserves versioned Stronghold FW configuration backups.

Its normal access boundary is limited to:

```text
authorized Stronghold FW appliances
local Net-Hunter administrator
```

Do not expose firewall configuration history to ordinary external UI/hunt users, the record-processing jail, or the PCAP-ingest jail.

Configuration history should be versioned/append-oriented rather than represented by one silently overwritten backup object.

The exact secret-handling and restore-authorization contract must be defined before configuration backup/restore implementation is treated as complete.

## State and Failure Discipline

Failure behavior is part of the product.

For capture-sensitive, durability-sensitive, destructive, privileged, authentication/trust-sensitive, clock-sensitive, or security-sensitive operations, implement and test failure paths at the same time as successful paths.

Examples include:

```text
packet drop
capture-ring pressure
short write
storage full
I/O error
fsync failure
process termination
power loss
segment finalization interruption
hash mismatch
compression failure
copy/transfer failure
Net-Hunter unavailable
destination full
authentication failure
LDAPS certificate validation failure
external authentication service unavailable
peer certificate failure
peer authorization/revocation failure
clock synchronization loss/large correction
verification failure
restart with interrupted artifacts
catalog/journal unavailable
record/index lag
jail unavailable
ZFS storage degradation
bridge failure
route application failure
firewall policy application failure
```

A failed verification/authentication/commit is still a factual result.

A partially completed operation must preserve what actually occurred.

Do not erase earlier failure history merely because a later retry succeeds.

## Durability Rules

`write()` success is not durability.

Do not report a durability-dependent Stronghold state until the required operating-system/filesystem durability boundary has completed.

Durability-sensitive implementation must be tested on the actual supported OS/filesystem/storage stack and representative hardware before Stronghold makes production durability claims.

Interrupted artifacts should be preserved when required for truthful reconciliation.

Cleanup must not destroy information required to determine what actually occurred.

## Storage and Migration Rules

On Stronghold FW, capture data begins on the fastest configured durable PCAP tier and may move to local backlog storage before transfer to Net-Hunter.

Only finalized closed segments may be migrated or transferred.

A source capture must not be deleted merely because a destination write or transfer completed. Destination finalization, integrity verification, history commit, and acknowledgement requirements must complete according to the applicable contract before source-retention state advances.

Compression, transfer, journal consolidation/transport, and other background work must throttle or pause when capture CPU, RAM-ring occupancy, packet-loss state, or storage I/O pressure indicates that capture needs the resources.

On Net-Hunter, ZFS protects storage, but ZFS state is not a replacement for Stronghold segment identities, hashes, source-journal identity/sequence, lineage, verification records, or transfer history.

## Secrets and Sensitive Material

Do not intentionally log/journal or persist plaintext credentials, private keys, passphrases, TOTP secrets/codes, or other authentication secrets outside an explicitly approved secure-storage design.

Packet captures may contain highly sensitive production content. Firewall configuration backups may also expose sensitive network architecture and policy information.

Debug logs, support bundles, tests, examples, fixtures, exports, and temporary workspaces must not accidentally include real capture data, credentials, customer information, or firewall configuration material.

## Repository Operations

Do not perform repository writes unless explicitly authorized for the specific action.

This includes commit, push, merge, pull-request creation, branch changes, ruleset/settings changes, issue creation, repository-content deletion, and other GitHub/repository writes.

Read-only repository inspection and local/offline working changes are permitted unless explicitly restricted.

Permission to create or modify local working files is not permission to commit or push them.

Permission for one repository write applies only to the specifically approved action/change set and does not carry forward automatically.

## Review Expectations

Before proposing a change as complete:

- compare the implementation against the applicable roadmap and architecture;
- verify Wireshark-class interface visibility has not been narrowed;
- verify capture priority has not been weakened;
- verify raw PCAP authority remains intact;
- verify journal domains have not been silently collapsed or made rewriteable;
- verify journal ordering/lineage remains explicit;
- verify packet-loss and degraded states remain truthful;
- verify records do not silently substitute inference for observation;
- verify physical observation remains distinguishable from authorization/bridge/route/firewall/NAT/WAN disposition;
- verify default-deny and authorization-before-routing remain intact;
- verify Policy ID is not used as rule priority;
- verify route selection and explicit WAN eligibility remain intact;
- verify AD authentication has not gained plaintext LDAP or downgrade behavior;
- verify authentication, authorization, and audit/journal responsibilities remain separate;
- verify stable appliance identity remains distinct from current certificate/IP/hostname;
- verify certificate validity does not silently grant peer authorization;
- verify timestamp resolution is not presented as accuracy;
- verify wall-clock correction cannot erase journal/event ordering;
- verify durability and transfer behavior remain explicit;
- verify Net-Hunter is not accidentally introduced into the live FW dependency path;
- verify External UI authoritative-data access remains read-only;
- verify FW configuration backup isolation remains intact;
- verify no future phase was accidentally pulled forward;
- verify security boundaries remain explicit;
- run all applicable checks defined by nested `AGENTS.md` files; and
- clearly report any check that could not be executed.

## Nested AGENTS.md Files

More specific directories may contain their own `AGENTS.md`.

A nested file may add or refine requirements for that portion of the tree but must not silently weaken repository-wide capture, durability, integrity, security, truthfulness, authorization, identity/trust, time, journal, Net-Hunter isolation, UI read-only, configuration-backup, or repository-write requirements.
