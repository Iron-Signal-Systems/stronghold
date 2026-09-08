# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation should work within the Stronghold repository.

It is an engineering map and behavioral contract. It does not replace the architecture and implementation contracts under `docs/`.

Stronghold is built around one governing principle:

> **Capture first. Never sacrifice observation for secondary work.**

Stronghold engineering must preserve truthful packet observation, explicit packet-loss accounting, durable capture semantics, clear failure behavior, operational simplicity, and the trust boundaries between Stronghold FW and Stronghold Net-Hunter.

## Product Model

Stronghold is a two-appliance system.

### Stronghold FW

Stronghold FW runs on Arch Linux and owns live-network responsibilities:

```text
observe
record
bridge and/or route
enforce
maintain local history backlog
transfer verified history to Net-Hunter
```

### Stronghold Net-Hunter

Stronghold Net-Hunter runs on FreeBSD with ZFS and jails and owns historical responsibilities:

```text
receive
verify
preserve
process
correlate
hunt
query
export
preserve FW configuration backups
```

Net-Hunter must never become a runtime dependency for Stronghold FW capture, bridging, routing, or firewall enforcement.

## Product Engineering Principles

### Capture first

Prefer work that improves, in order:

1. Wireshark-class visibility on configured physical interfaces;
2. packet capture correctness;
3. truthful loss accounting;
4. durable local capture;
5. integrity;
6. comprehensive records;
7. explainability;
8. failure behavior;
9. operational simplicity; and
10. downstream processing.

The first capture requirement is explicit:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

A supported capture path should provide the same class of interface visibility expected from Wireshark/dumpcap operating on the same supported physical interface under the same conditions.

Unknown EtherTypes, unknown IP protocols, malformed traffic, vendor-specific frames, and non-routable Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

Live packet capture and active durable PCAP writes take priority over transfer, compression, indexing, analytics, hunt activity, and other background work.

Secondary work may throttle, pause, or fall behind. Stronghold must not deliberately sacrifice live capture merely to keep secondary systems current.

### Raw packet history is authoritative

Raw PCAPNG is the authoritative packet history.

Structured observation, flow, protocol, bridge, firewall, routing, NAT, system, and failure records exist to explain, correlate, locate, and make the authoritative packet history usable.

Failure to recognize, decode, enrich, index, or correlate traffic must not cause the underlying packet to be discarded.

Absence of a decoded record is not proof that the corresponding traffic did not exist.

### Observation and disposition are different facts

Stronghold must preserve the distinction between what a physical interface observed and what the appliance later did with the traffic.

Do not collapse ingress observation, bridge forwarding, route selection, firewall action, NAT, egress interface, and egress VLAN into one ambiguous event.

Router-on-a-stick traffic may enter and leave the same physical interface under different VLAN contexts. Preserve the correlation without pretending the observations are unrelated traffic.

A DROP or REJECT disposition must not by itself suppress authoritative ingress capture.

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

Packet drops, capture gaps, record/index lag, Net-Hunter transfer backlog, storage pressure, and verification failures must remain observable.

### Simple does not mean vague

Stronghold should remain operationally simple while retaining authoritative technical detail.

Do not hide Linux, FreeBSD, NIC, packet-capture, bridge, routing, firewall, filesystem, ZFS, jail, storage, or failure behavior behind vague abstractions.

### Prefer explicit engineering

Prefer simple, narrow, inspectable implementation over speculative abstractions.

Do not create generic plugin frameworks, arbitrary execution systems, catch-all platform frameworks, generalized registries/factories, or unnecessary service boundaries for anticipated future needs.

Implement the requirement that exists now.

## Sources of Truth

Read the applicable documents before changing implementation behavior.

### Phase scope and sequence

`docs/ROADMAP.md` defines current implementation phase scope, sequencing, targets, and exit gates.

The broader architecture may be documented before its implementation phase begins. Do not implement future systems merely because they are already described architecturally.

### Architecture invariants

`docs/ARCHITECTURE.md` defines the current Stronghold FW, Net-Hunter, capture, networking, storage, history-transfer, jail, and trust-boundary architecture.

Future dedicated contracts may refine implementation details without silently weakening these invariants.

## Scope Discipline

Work only within the current roadmap phase and current engineering slice unless explicitly directed otherwise.

Stronghold begins implementation with the FW capture foundation. Do not pull Layer-2 bridging, routing, nftables enforcement, FQDN policy, VPN, IDS/IPS, external UI implementation, Net-Hunter processing, HA, dynamic routing, or other future systems into the current phase unless explicitly approved.

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

Do not silently weaken capture completeness claims, packet-loss accounting, durability requirements, record completeness, interface/VLAN identity, forwarding-disposition traceability, integrity verification, storage migration safety, Net-Hunter transfer verification, UI read-only boundaries, configuration-backup isolation, retention behavior, or lineage/provenance.

## Stronghold FW Rules

### Capture Requirement #1 — Wireshark-class interface visibility

The authoritative capture point is the configured supported physical interface.

If traffic is presented to that interface and observable through the supported NIC/driver capture path, Stronghold records it without first requiring protocol recognition, decoding, routability, bridge relevance, or firewall relevance.

This includes, when presented to the interface, ordinary IP traffic and Layer-2/control-plane traffic such as ARP, DHCP, DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes, unknown IP protocols, vendor-specific frames, and malformed traffic.

Stronghold must not claim to have observed traffic that the NIC, upstream topology, hardware filtering, or driver did not present to the supported capture path.

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline for capture validation.

The authoritative configured capture stream is not filtered merely because downstream retention, transfer, forwarding, or processing selects a subset of traffic.

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

Connected and static routing are the initial intended routing direction. Dynamic routing remains future work unless explicitly approved.

### FW network objects

VLAN is a first-class Stronghold object.

Do not represent VLAN meaning only as an anonymous integer buried in a Linux subinterface name.

Records and configuration must preserve both Stronghold object identity and the actual 802.1Q VLAN ID.

The architecture also anticipates explicit physical-interface, logical-interface, bridge-domain, zone, host, network, address-group, service, service-group, FQDN, and FQDN-group objects. Exact schemas must be defined before implementation is treated as complete.

### FQDN truthfulness

DNS-backed FQDN policy and directly observed Layer-7 hostname identity are different facts.

Do not claim that a connection used a hostname merely because its destination IP appeared in a DNS resolution set.

When Layer-7 hostname information is later supported, preserve the source of that knowledge, such as TLS SNI, HTTP Host, DNS correlation, or `not_observed`.

### FW storage direction

The current preferred FW storage model is:

```text
separate SSD / Btrfs direction for Arch Linux OS
NVMe / XFS direction for HOT PCAP capture
local SSD / XFS direction for WARM/backlog PCAP
Net-Hunter for long-term history
```

PCAP storage must not share the root filesystem in a way that allows capture exhaustion to silently exhaust the operating system.

Long-term HDD/RAID history belongs to Net-Hunter rather than the live firewall.

### Net-Hunter independence

Net-Hunter unavailability must not stop Stronghold FW capture, bridging, routing, or firewall enforcement while local FW resources remain capable of operating.

A Net-Hunter outage creates explicit transfer backlog/degraded state.

Do not silently discard pending history or claim successful transfer when Net-Hunter has not verified and acknowledged it.

## Dedicated History Network Rules

Stronghold FW and Net-Hunter use a dedicated history-transfer path.

Current physical-link direction includes 10 GbE, 25 GbE, and 40 GbE interfaces as deployment options.

History-link speed is independent of the validated Stronghold FW dataplane rating. Do not infer or advertise a firewall forwarding/capture capability from the speed of the dedicated history interface.

History transfer is a verified handoff, not a blind copy/delete operation.

A source segment must remain distinguishable from a transfer acknowledgement. Destination receipt, finalization, verification, commit, and acknowledgement are separate facts unless a future contract intentionally combines specific steps.

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

The ingest jail receives finalized PCAP segments and associated initial records from authorized Stronghold FW appliances.

It may perform source authentication, transfer-completeness checks, integrity verification, commit work, and verified receipt acknowledgement.

It does not provide external hunt/query user access.

### Record Processing Jail

The record-processing jail reads authoritative PCAP and initial FW records and produces/enriches searchable records and indexes.

It may reprocess historical PCAP when future decoders improve.

Normal record processing must not rewrite authoritative packet history.

### External User Interface Jail

The External UI jail is read-only with respect to authoritative Stronghold traffic/history data.

It may:

```text
hunt
query
search
correlate
build timelines
view records
retrieve referenced PCAP
create controlled derived exports
create reports
view status
```

It MUST NOT:

```text
modify authoritative PCAP
modify observation records
modify processed records
rewrite indexes
delete historical data
change ingest state
change retention policy
modify FW configuration backups
modify Stronghold FW policy/configuration through the hunt interface
```

Writable UI storage is limited to non-authoritative material such as session state, temporary query work, derived PCAP exports, reports, and download staging.

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

For capture-sensitive, durability-sensitive, destructive, privileged, or security-sensitive operations, implement and test failure paths at the same time as successful paths.

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
verification failure
restart with interrupted artifacts
catalog unavailable
record/index lag
jail unavailable
ZFS storage degradation
bridge failure
route application failure
firewall policy application failure
```

A failed verification is still a factual verification result.

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

Compression, transfer, and other background work must throttle or pause when capture CPU, RAM-ring occupancy, packet-loss state, or storage I/O pressure indicates that capture needs the resources.

On Net-Hunter, ZFS protects storage, but ZFS state is not a replacement for Stronghold segment identities, hashes, lineage, verification records, or transfer history.

## Secrets and Sensitive Material

Do not intentionally log or persist plaintext credentials, private keys, passphrases, or other authentication secrets outside an explicitly approved secure-storage design.

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
- verify packet-loss and degraded states remain truthful;
- verify records do not silently substitute inference for observation;
- verify physical observation remains distinguishable from bridge/route/firewall/NAT disposition;
- verify VLAN and interface identities are preserved explicitly;
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

A nested file may add or refine requirements for that portion of the tree but must not silently weaken repository-wide capture, durability, integrity, security, truthfulness, networking identity, Net-Hunter isolation, UI read-only, configuration-backup, or repository-operation requirements.

Current nested engineering standard:

```text
go/AGENTS.md
    -> applies to the complete Go module tree
```
