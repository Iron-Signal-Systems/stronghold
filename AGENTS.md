# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation should work within the Stronghold repository.

It is an engineering map and behavioral contract. It does not replace the architecture and implementation contracts under `docs/`.

Stronghold is built around one governing principle:

> **Capture first. Never sacrifice observation for secondary work.**

Stronghold engineering must preserve truthful packet observation, explicit packet-loss accounting, durable capture semantics, clear failure behavior, and operational simplicity.

## Product Engineering Principles

### Capture first

Prefer work that improves, in order:

1. Wireshark-class visibility on configured physical interfaces;
2. packet capture correctness;
3. truthful loss accounting;
4. durable local capture;
5. integrity;
6. explainability;
7. failure behavior;
8. operational simplicity; and
9. downstream processing.

The first capture requirement is explicit:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, routes, or firewall-processes it.**

A supported capture path should provide the same class of interface visibility expected from Wireshark/dumpcap operating on the same supported physical interface under the same conditions.

Unknown EtherTypes, unknown IP protocols, malformed traffic, vendor-specific frames, and non-routable Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

Live packet capture and active durable PCAP writes take priority over compression, tier migration, remote offload, indexing, analytics, and other background work.

Secondary work may throttle, pause, or fall behind. Stronghold must not deliberately sacrifice live capture merely to keep secondary systems current.

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

Packet drops, capture gaps, index lag, offload lag, storage pressure, and verification failures must remain observable.

### Simple does not mean vague

Stronghold should remain operationally simple while retaining authoritative technical detail.

Do not hide Linux behavior, packet-capture limitations, storage state, or failure conditions behind vague abstractions.

### Prefer explicit engineering

Prefer simple, narrow, inspectable implementation over speculative abstractions.

Do not create generic plugin frameworks, arbitrary execution systems, catch-all platform frameworks, generalized registries/factories, or unnecessary service boundaries for anticipated future needs.

Implement the requirement that exists now.

## Sources of Truth

Read the applicable documents before changing implementation behavior.

### Phase scope and sequence

`docs/ROADMAP.md` defines current phase scope, sequencing, implementation targets, and exit gates.

Do not implement future roadmap phases merely because their architecture has already been discussed.

### Architecture invariants

`docs/ARCHITECTURE.md` defines the current capture, storage, tiering, offload, and resource-priority architecture.

Future dedicated contracts may refine implementation details without silently weakening these invariants.

## Scope Discipline

Work only within the current roadmap phase and current engineering slice unless explicitly directed otherwise.

Stronghold begins as a capture platform. Do not pull routing, nftables enforcement, FQDN policy, VPN, IDS/IPS, web UI, HA, dynamic routing, or other future systems into Phase 0 unless explicitly approved.

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

Do not silently weaken capture completeness claims, packet-loss accounting, durability requirements, integrity verification, storage migration safety, retention behavior, or offload provenance.

## Capture Rules

### Capture Requirement #1 — Wireshark-class interface visibility

The authoritative capture point is the configured supported physical interface.

If traffic is presented to that interface and observable through the supported NIC/driver capture path, Stronghold records it without first requiring protocol recognition, decoding, routability, or firewall relevance.

This includes, when presented to the interface, ordinary IP traffic and Layer-2/control-plane traffic such as ARP, DHCP, DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes, unknown IP protocols, vendor-specific frames, and malformed traffic.

Stronghold must not claim to have observed traffic that the NIC, upstream topology, hardware filtering, or driver did not present to the supported capture path.

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline for Phase 0 validation.

The authoritative configured capture stream is not filtered merely because downstream retention or offload selects a subset of traffic.

For example, if Stronghold is configured to capture `eth0` and `eth1` and offload VLANs 1, 2, and 201, Stronghold still captures the complete configured local stream first. VLAN selection is a downstream retention/offload decision.

RAM is buffering only and is not durable capture storage.

The initial preferred packet-acquisition direction is Linux AF_PACKET with TPACKET_V3 unless measurement or a later approved contract requires another source.

Any capture-source abstraction must be justified by a real implementation boundary rather than speculative portability.

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
copy failure
destination full
offload destination unavailable
credential failure
restart with interrupted artifacts
catalog unavailable
catalog lag
```

A failed verification is still a factual verification result.

A partially completed operation must preserve what actually occurred.

Do not erase earlier failure history merely because a later retry succeeds.

## Durability Rules

`write()` success is not durability.

Do not report a durability-dependent Stronghold state until the required Linux/filesystem durability boundary has completed.

Durability-sensitive implementation must be tested on the actual supported Linux/filesystem/storage stack and representative hardware before Stronghold makes production durability claims.

Interrupted artifacts should be preserved when required for truthful reconciliation.

Cleanup must not destroy information required to determine what actually occurred.

## Storage Tiering Rules

The preferred capture lifecycle is downward:

```text
RAM buffer
    -> NVMe hot tier
    -> SSD warm tier
    -> HDD/RAID cold tier
    -> optional archive/offload
```

Capture segments are written to the fastest configured durable ingest tier first. Traffic load changes residence time and background-drain behavior, not the primary capture path.

Only finalized closed segments may be migrated downward.

A source capture must not be deleted merely because a destination write completed. The destination must first be finalized and verified according to the applicable contract.

Compression belongs to downward tiering, not live capture. Compression must throttle or pause when capture CPU, RAM-ring occupancy, packet-loss state, or storage I/O pressure indicates that capture needs the resources.

## Offload Rules

Local capture is authoritative for the configured capture stream.

Remote SMB, SFTP, SSH, iSCSI-backed, or other archive availability must not determine whether local capture continues.

Derived/offloaded captures must preserve provenance back to their source segment(s) and selection criteria.

Do not silently claim a remote copy is complete until transfer and destination verification requirements have completed.

## Secrets and Sensitive Material

Do not intentionally log or persist plaintext credentials, private keys, passphrases, or other authentication secrets outside an explicitly approved secure-storage design.

Packet captures may contain highly sensitive production content. Debug logs, support bundles, tests, examples, and fixtures must not accidentally include real capture data, credentials, or customer information.

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
- verify that capture priority has not been weakened;
- verify packet-loss and degraded states remain truthful;
- verify durability and migration behavior remain explicit;
- verify no future phase was accidentally pulled forward;
- verify security boundaries remain explicit;
- run all applicable checks defined by nested `AGENTS.md` files; and
- clearly report any check that could not be executed.

## Nested AGENTS.md Files

More specific directories may contain their own `AGENTS.md`.

A nested file may add or refine requirements for that portion of the tree but must not silently weaken repository-wide capture, durability, integrity, security, truthfulness, or repository-operation requirements.

Current nested engineering standard:

```text
go/AGENTS.md
    -> applies to the complete Go module tree
```
