# Stronghold

**Stronghold by Iron Signal Systems**

Stronghold is an enforcement system that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.

Stronghold is built as two cooperating appliances:

- **Stronghold FW** runs on Arch Linux and owns live observation, preservation, authorization, bridging/routing, NAT, enforcement, journaling, local backlog, and optional active/standby HA.
- **Stronghold Net-Hunter** runs on FreeBSD with ZFS and jails and owns verified historical ingest, durable preservation, processing/reprocessing, correlation, hunt/query, export, retention/holds, and isolated FW configuration backup.

```text
OBSERVE
   ↓
RECORD
   ↓
ENFORCE
   ↓
REMEMBER
   ↓
HUNT
```

> **Observe the truth. Preserve the history. Explain the decision.**

## Stronghold Truth Model

Stronghold separates three categories of truth:

```text
WHAT WAS PRESENTED
    authoritative physical-interface observation / PCAPNG

WHAT STRONGHOLD DID
    authoritative operational and decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation, correlation, and enrichment
```

Observation survives interpretation. A packet Stronghold cannot decode today remains authoritative history if it was actually captured. Better future decoders may create new derived interpretation without rewriting the original observation.

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **History is journaled, not casually logged.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

> **Stronghold owns the appliance operating-system and release lifecycle.**

> **Record what the system can actually establish, preserve the source of that truth, and never manufacture certainty beyond it.**

## Common Truth Separations

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
key rotated                     != past exposure undone
entry written                   != journal entry durably committed
signed journal checkpoint       != physically immutable storage
system booted                   != approved system state
hardware detected               != hardware supported
link speed                      != validated dataplane rate
capture throughput              != full-feature firewall throughput
node performance                != HA cluster performance
storage capacity                != sustainable ingest capacity
AF_XDP available                != AF_XDP zero-copy available
zero-copy available             != Stronghold zero-copy qualified
```

## Capture Requirement #1 — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Physical-interface frame observation is a product invariant of the enforcement appliance itself. Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline.

Protocol recognition is not required before capture. Observable IPv4/IPv6, ARP, DHCP/DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes/IP protocols, vendor-specific frames, and malformed traffic remain in scope when presented to the supported capture path.

Stronghold does not claim visibility into traffic that topology, NIC hardware, filtering/offload behavior, or the driver never presents.

Stronghold FW uses **AF_XDP as the intended packet-acquisition foundation**. AF_XDP is subordinate to the capture invariant: the selected XDP/AF_XDP mode, queue/RSS behavior, UMEM/ring design, NIC/driver/firmware behavior, and CPU/NUMA/PCIe placement must be qualified rather than assumed. See [`docs/AF-XDP.md`](docs/AF-XDP.md).

## Stronghold FW

Current platform direction:

```text
Arch Linux / x86_64
CLI-first appliance
physical qualified PCIe Ethernet NICs
AF_XDP packet-acquisition foundation
continuous PCAPNG
nftables enforcement foundation
Btrfs OS storage
XFS NVMe HOT capture
XFS SSD WARM/backlog
1 GbE current qualification target
10 GbE primary production target
40 GbE future target only
```

Stronghold supports Layer-2 transparent bridging, Layer-3 IPv4/IPv6 routing, router-on-a-stick over 802.1Q trunks, and hybrid deployments. VLANs, logical interfaces, bridge domains, zones, routes, NAT, WAN preference, and policy are explicit Stronghold objects rather than hidden Linux configuration.

### Packet-processing model

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

A route lookup never grants authorization. Stronghold is explicit-allow/default-deny. Policy positions are dense mutable integers evaluated lowest-to-highest, first match wins. **Policy ID is stable identity only, not priority.**

Route selection is longest-prefix match first, then equal-prefix source preference `STATIC`, `CONNECTED`, `DEFAULT`, `DYNAMIC`, followed by health, metric, and deterministic tie-break. NAT never grants permission. Multi-WAN chooses only among paths explicitly authorized for the destination/service.

## Stronghold FW High Availability

Initial HA is optional **active/standby**, not active/active.

Each node keeps its own Appliance ID, physical NIC/MAC identity, management/history/HA identity, local storage, clock state, certificates, source journals, and capture provenance. The pair has a stable Cluster ID and cluster-owned virtual forwarding identity.

> **Virtual network identity is forwarding identity, not capture identity.**

Cluster configuration and node-local configuration remain distinct. Heartbeat/control and bulk state synchronization are logically separate. Bulk synchronization may lag; heartbeat/control must not be starved; capture remains the highest live-data priority.

> **Loss of peer communication is not, by itself, proof that the peer is dead.**

> **A standby must not claim ACTIVE cluster identity until the fencing/election contract permits promotion.**

Net-Hunter is not an HA witness/quorum dependency. Successful failover never justifies a claim of zero capture gap.

## Stronghold-Controlled Appliance Updates

Stronghold FW is an appliance built on Arch Linux, not a supported general-purpose rolling Arch host. Stronghold-qualified releases bind and validate relevant software, kernel, NIC driver/firmware, nftables/netfilter, capture stack, configuration schema, journal schema, and HA protocol/state compatibility.

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
READY
```

For HA, update the standby first, validate it, perform controlled failover, validate production behavior, then update the former active. A failed standby update stops the rolling process; Stronghold does not automatically risk the remaining healthy node.

Signed offline update bundles are an architectural requirement for restricted environments. A successful package install or boot is not sufficient update validation.

## Recovery and Disaster Recovery

Stronghold distinguishes:

```text
REBUILDABLE PLATFORM
RECOVERABLE CONFIGURATION
APPLIANCE / TRUST IDENTITY
AUTHORITATIVE HISTORY
DERIVED STATE
TRANSIENT RUNTIME STATE
```

FW configuration backups are versioned in the isolated Net-Hunter FW Configuration Backup Jail. Configuration backup is separate from protected secret/private-key recovery.

Hardware replacement may preserve the logical Stronghold Appliance ID only through an explicit privileged recovery operation. Replacement normally generates new cryptographic credentials and re-establishes authorized trust rather than cloning old private keys.

Physical-interface mappings must be revalidated on replacement hardware before activation. Old runtime conntrack/NAT/session state is not blindly restored; HA nodes synchronize current runtime state from the active peer where applicable.

Surviving failed-node PCAP storage may be recovered offline. Recovered PCAP retains the original source Appliance ID and observation time and is recorded as recovered history, not as a new observation.

For Net-Hunter, authoritative PCAP/journals/configuration/hold state are fundamentally different from rebuildable indexes and derived records. ZFS RAID/checksums/snapshots are protection mechanisms, not disaster recovery.

After major Hunter recovery, destructive retention remains disabled until authoritative history, journals/lineage, trust state, active holds, and retention state are validated.

Missing authoritative history is recorded as a permanent gap rather than silently healed.

## Encryption and Key Management

Stronghold protects authoritative packet history, journals, configuration history, and sensitive appliance state using purpose-separated encryption domains.

Current direction:

```text
Stronghold FW
    encrypted block devices beneath Btrfs/XFS
    TPM 2.0 anticipated for normal key protection
    separate controlled recovery authority

Net-Hunter
    ZFS-native encrypted datasets aligned with host/jail authority
    separate encryption domains for authoritative history,
    journals, derived/index state, and FW config backups
```

TPM/HSM-backed keys may strengthen normal operation but must not be the sole recovery path for authoritative history. HA membership does not imply shared local disk keys.

Private appliance identity keys should be non-exportable where supported. Hardware replacement normally generates new private credentials while preserving the authorized logical Appliance ID.

Routine key rotation should not inherently require rewriting all historical PCAP. Key lifecycle and recovery actions are journaled. Crypto-shredding, if ever supported, is a privileged destructive action subject to hold/destruction authority and is not ordinary retention cleanup.

## Cryptographic Journal Integrity

Stronghold journals are separate append-oriented, cryptographically tamper-evident streams:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Each journal has its own identity, epoch, monotonic local sequence, hash-linked committed entries, and canonical exact-byte representation for integrity calculation.

Periodic signed checkpoints/finalized journal segments authenticate journal heads without requiring an asymmetric signature for every event. Wall-clock time does not establish journal order; sequence does.

Net-Hunter independently verifies FW journal chains/checkpoints and may preserve externally known FW heads without becoming a runtime requirement for FW journaling. Hunter's Processing Journal remains a separate authority and chain.

A continuity break creates an explicit gap/new epoch rather than a silently repaired chain. Authorized retention may remove old journal content only while preserving enough checkpoint/tombstone lineage to show that the range existed and was deliberately destroyed.

Stronghold describes this as **append-oriented and cryptographically tamper-evident**, not physically immutable unless a later storage/witness mechanism justifies that stronger claim.

## Hardware Trust and Platform State

Stronghold distinguishes:

```text
logical Appliance ID
hardware/root-of-trust identity
certificate identity
platform trust state
```

Secure Boot, measured boot, and attestation are separate capabilities. TPM-backed measured/sealed operation is an anticipated FW direction, but TPM state is not the sole recovery authority.

A future platform-trust state may distinguish `VERIFIED`, `DEGRADED`, `UNVERIFIED`, `MISMATCH`, and `RECOVERY`. An HA node with unacceptable platform-trust state is not silently considered a healthy automatic promotion target.

Development hardware, qualification hardware, and supported production hardware are distinct categories.

## Hardware Qualification

Stronghold support/performance claims apply to **qualified appliance profiles**, not arbitrary systems capable of booting the software.

Qualification binds Stronghold release, CPU, NIC/driver/firmware, PCIe topology, NUMA behavior where applicable, RAM, storage, platform trust, enabled feature set, and the qualified XDP/AF_XDP operating mode.

NIC qualification covers capture-visible behavior including multi-queue/RSS, AF_XDP queue binding, UMEM/ring behavior, copy/zero-copy mode, VLAN/checksum/offload representation, filtering/promiscuous behavior, ring/drop behavior, MTU, driver resets, and firmware revision.

Performance qualification measures both bandwidth and packet rate across representative packet sizes and traffic profiles. Capture-only throughput is not used as a claim for full stateful/NAT/HA feature sets.

FW qualification includes sustained capture/write behavior and truthful NIC/XDP/AF_XDP/user-space/writer/intentional-policy drop accounting. HA qualification measures node-pair behavior, fencing/promotion, session/state continuity, virtual identity convergence, forwarding interruption, capture gap, and journal truthfulness separately.

Net-Hunter qualification separates capacity from sustainable ingest, verification, indexing, query, reprocessing, retention, scrub/resilver, and degraded-storage behavior. ECC remains the required direction for Net-Hunter and a strong production preference for FW until exact supported profiles are frozen.

## Packet History and Net-Hunter

FW local history path:

```text
RAM buffer
    ↓
NVMe / XFS HOT
    ↓
SSD / XFS WARM / backlog
    ↓
dedicated 10/25/40 GbE HISTORY network
    ↓
Net-Hunter
```

Verified handoff is not `copy then delete`:

```text
FW finalizes object
    ↓
integrity established
    ↓
mTLS transfer to explicitly authorized Hunter
    ↓
Hunter receives/finalizes
    ↓
independent verification
    ↓
durable commit
    ↓
Hunter Processing Journal
    ↓
ACK
    ↓
FW records acknowledgement
```

Hunter never ACKs unverified/uncommitted history.

Net-Hunter currently has four defined application jails:

1. **PCAP Data Ingest Jail**
2. **Record Processing Jail**
3. **External User Interface Jail**
4. **FW Configuration Backup Jail**

The External UI is read-only with respect to authoritative PCAP, source journals, processed authoritative history/indexes, and FW configuration history. Writable UI state is non-authoritative workspace only.

## Retention, Holds, Archive, and Destruction

PCAP and each journal domain have independent retention policy.

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

Holds override ordinary expiration. Manual destruction is privileged and attributable. Archive is not destruction. Storage pressure may throttle secondary work and accelerate safe transfer/tiering, but it never invents deletion authority or permits Hunter to ACK uncommitted history.

## Current Scope and Explicit Deferrals

Implementation begins with the Stronghold FW **Phase 0 Traffic Observation Foundation**. AF_XDP is the Phase 0 packet-acquisition foundation. Routing, firewall enforcement, HA, complete journal integrity, recovery/DR, encryption/key management, production platform trust, and complete appliance update orchestration are later implementation work even though their architectural direction is documented now.

**VPN is explicitly deferred. IDS/IPS is explicitly deferred.** Neither is part of Phase 0, an early performance claim, or a dependency of the current capture architecture. Nothing built now should prevent future detection from consuming authoritative PCAP and producing derived results, but no IDS/IPS engine, inline behavior, TLS interception, ruleset model, or IPS enforcement contract is being selected now.

## Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

If it cannot, the feature is not complete.

## Engineering Standard

Stronghold follows the Iron Signal Systems Engineering Standards pinned by `ENGINEERING-STANDARD`. Contributor behavior is governed by `AGENTS.md` and nested `AGENTS.md` files.

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md), [`docs/AF-XDP.md`](docs/AF-XDP.md), and [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Project Status

Stronghold is pre-release and under active development. Architecture and interfaces may change before a supported release.

## License

Stronghold is proprietary source-available software. See [`LICENSE`](LICENSE).

## Security

Do not report suspected vulnerabilities through public GitHub issues. See [`SECURITY.md`](SECURITY.md).
