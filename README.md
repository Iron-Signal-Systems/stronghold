# Stronghold

**Stronghold by Iron Signal Systems**

Stronghold is a two-appliance network security system built around complete network observation, comprehensive records, policy enforcement, retained packet history, and durable operational journals.

**Stronghold FW** runs on Arch Linux and observes, records, authorizes, bridges or routes, and enforces live traffic.

**Stronghold Net-Hunter** runs on FreeBSD with ZFS and jails and receives verified packet history and journals, processes network records, preserves historical PCAP, and provides a read-only hunt/query interface.

The system is built around five responsibilities:

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

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

Live packet capture and durable local writes take priority over transfer, compression, indexing, analytics, hunt activity, and other background work.

> **Denied traffic should be cheap to reject, but never invisible.**

A packet does not earn routing, NAT, or deeper forwarding work merely because it is technically routable. Stronghold observes available traffic first, then requires policy authorization before normal forwarding work proceeds.

> **Path availability is not path permission.**

Stronghold never assumes that a destination or service may use an alternate WAN merely because that WAN is physically available.

> **History is journaled, not casually logged.**

Administrative actions, system/health state, traffic decisions, trust changes, clock events, and Net-Hunter processing activity are preserved in separate append-oriented journals. Corrections and superseding state create new entries rather than silently rewriting prior history.

> **Expiration eligibility is not permission to delete.**

Retention is policy-driven and separate for PCAP and each journal domain. Holds override ordinary expiration, and destruction is explicit, attributable, and journaled.

## Capture Requirement #1 — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

The intended visibility baseline is Wireshark/dumpcap on the same supported physical interface under the same conditions.

Protocol recognition is not required before capture. When presented to the interface, Stronghold should preserve ordinary IPv4/IPv6 traffic as well as Layer-2/control-plane traffic such as ARP, DHCP, DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes, unknown IP protocols, vendor-specific frames, and malformed traffic.

Stronghold does not claim visibility into traffic that the network topology, NIC hardware, hardware filtering/offload behavior, or driver never presents to the supported capture path.

## Stronghold FW

Current platform direction:

- Arch Linux, minimal CLI-focused appliance installation;
- x86_64;
- physical PCIe Ethernet interfaces with strong Linux driver support;
- 1 GbE and 10 GbE as current dataplane targets;
- 40 GbE preserved as a future dataplane hardware target only and not currently claimed as tested or supported;
- Linux-native packet capture;
- AF_PACKET / TPACKET_V3 as the initial capture direction;
- continuous full-packet PCAPNG capture;
- local-first durable packet storage;
- Layer-2 transparent bridging where configured;
- Layer-3 IPv4/IPv6 routing where configured;
- router-on-a-stick operation over 802.1Q trunks;
- hybrid deployments in which different interfaces, VLANs, or bridge domains use different forwarding models; and
- nftables as the expected native Linux enforcement foundation beneath Stronghold policy.

### FW packet-processing model

Stronghold policy semantics are intentionally authorization-first:

```text
PACKET / FRAME ARRIVES
        │
        ├──────────────► CAPTURE / OBSERVE
        │
        ▼
EARLY INGRESS AUTHORIZATION
        │
        ├── explicit DENY ───────────────► DROP
        ├── no permitting match ─────────► DROP
        └── authorized to proceed
                    │
                    ▼
               ROUTING
                    │
                    ▼
        NAT / STATE / REQUIRED
           DEEPER PROCESSING
                    │
                    ▼
              FINAL EGRESS
```

For future Layer-7 policy, an early authorization may mean only that a packet/session is permitted to consume the processing needed to reach a final decision. Stronghold must not report a final Layer-7 allow before the required application-layer fact has actually been established.

### Default-deny policy

Stronghold is explicit-allow and default-deny for traffic subject to Layer-2 forwarding policy, Layer-3/4 forwarding policy, and traffic destined to the appliance itself.

Rules are evaluated from the lowest current rule position to the highest. The first matching rule determines the action. Explicit deny rules may be placed early so unwanted traffic can be rejected before unnecessary downstream work.

Rule positions are dense mutable integers:

```text
1
2
3
...
N
```

Inserting a rule at an occupied position pushes the existing rule and all following rules down while preserving their relative order.

A **Policy ID is a stable reference identity only**. It does not determine rule priority or evaluation order.

### FW networking model

Stronghold does not require the appliance to operate in one global forwarding mode.

```text
Layer 2 transparent
    bridge Ethernet/VLAN traffic without becoming the IP gateway

Layer 3 routed
    terminate IP networks and route between them

Router on a stick
    terminate and route multiple VLANs over one physical 802.1Q trunk

Hybrid
    use Layer 2 bridging and Layer 3 routing on different interfaces,
    VLANs, or bridge domains on the same Stronghold FW
```

VLANs are first-class Stronghold objects rather than anonymous numeric values embedded only in Linux interface names. Stronghold records preserve both the Stronghold object identity and the actual VLAN ID.

The broader object model is expected to include physical interfaces, logical interfaces, VLANs, bridge domains, zones, hosts, networks, address groups, services, service groups, FQDNs, and FQDN groups.

FQDN policy is expected to distinguish DNS-resolved address-set policy from future true Layer-7 hostname observation. Stronghold must not treat an IP obtained from DNS as proof that a specific connection actually carried that hostname.

### Routing

Only authorized traffic enters normal Stronghold routing.

Route selection follows:

```text
1. longest-prefix match
2. for equal-prefix candidates, route-source preference:
       STATIC
       CONNECTED
       DEFAULT
       DYNAMIC
3. health / availability
4. metric
5. deterministic tie-break
```

A more-specific prefix always wins over a less-specific prefix. For example, `10.12.12.3/32` wins over `10.12.12.0/24` for destination `10.12.12.3` regardless of route-source preference.

VRF is an anticipated future capability, not an initial requirement. Initial routing uses one default routing domain, while interfaces, routes, policy, and records should retain a routing-domain concept so VRFs can be added later without redesigning the model.

### Multi-WAN preference

Stronghold uses **policy-defined WAN preference**, not generic traffic spraying or packet-level balancing.

A destination or service may explicitly authorize one or more WANs and identify a preferred WAN. Scheduled preference and adaptive path selection may choose among only those explicitly authorized WANs.

Adaptive selection may consider destination-specific path quality such as reachability, round-trip time, loss, and jitter, with thresholds and hysteresis to prevent path flapping. Existing established sessions normally remain bound to their established WAN; adaptive changes apply to new eligible flows unless the existing path actually fails.

Stronghold never assumes that a physically available WAN is authorized for a site or service.

### Zones and interface roles

Zones represent security boundaries. Interfaces and VLANs represent actual network attachment.

One routed logical interface belongs to one security zone; one zone may contain multiple logical interfaces or VLANs.

Current interface-role model:

```text
PRODUCTION
WAN
MANAGEMENT
HISTORY
```

`MANAGEMENT` and `HISTORY` interfaces are not ordinary production transit paths. Configuration validation must reject unsafe combinations such as a production default route through the history interface or placing the management interface in a production bridge domain.

### FW configuration model

Stronghold uses a transactional configuration model:

```text
RUNNING CONFIGURATION
        ↓
CANDIDATE CONFIGURATION
        ↓
VALIDATE
        ↓
SHOW DIFF
        ↓
COMMIT
        ↓
NEW CONFIGURATION GENERATION
```

Each committed generation represents the complete effective Stronghold configuration, including interfaces, VLANs, bridge domains, zones, routes, WAN preference, NAT, security policy, objects, management settings, and Hunter settings as applicable.

Risky remote changes should support commit-confirmed protection with automatic rollback if the new management path is not confirmed.

Rollback restores the content of a prior generation by creating a **new** generation; Stronghold does not rewrite historical generation identity.

Successful committed generations are intended to be versioned in the isolated Net-Hunter FW Configuration Backup Jail.

## Administrative Identity and Authorization

Stronghold separates authentication, authorization, and journaling of administrative activity.

Normal remote authentication may use:

```text
Active Directory via LDAPS only
RADIUS
TACACS+
```

For Active Directory, plaintext LDAP is not supported for authentication and Stronghold must never downgrade from LDAPS to LDAP because secure authentication is unavailable. LDAPS certificate validation is required.

Protected local appliance identity remains available for installation, physical-console recovery, and explicit break-glass use. External authentication failure must not affect capture, routing, or firewall enforcement.

Stronghold uses role-based authorization and keeps important privileges separable, including:

```text
view configuration
edit candidate configuration
validate candidate configuration
commit configuration
rollback configuration
system administration
hunt/query history
export packet history
```

Viewing Hunter history does not imply permission to export packet data, and access to Net-Hunter does not imply authority to administer Stronghold FW.

Normal remote administration is expected to enter through the `MANAGEMENT` role/interface and may be restricted to explicitly configured management source networks/hosts. The `HISTORY` interface is not a normal administrative login path.

## Appliance Identity and History-Link Trust

Stronghold FW and Net-Hunter mutually authenticate across the dedicated history network using mTLS.

Each appliance has a stable Stronghold Appliance ID independent of hostname, IP address, or the individual certificate currently used to authenticate it.

A valid certificate proves cryptographic identity; it does **not** automatically authorize a FW to send history to a Hunter. FW↔Hunter relationships are explicitly authorized and revocable.

The architecture favors a Stronghold-specific appliance trust hierarchy rather than coupling history-link appliance identity directly to an organization's Active Directory PKI. Exact CA topology and enrollment mechanics remain to be frozen.

Certificate rotation preserves the stable Appliance ID. Certificate expiry, revocation, or peer-authorization failure blocks history transfer and creates explicit backlog/degraded state, but does not stop the FW dataplane while local resources remain available.

Management/UI certificates and history-transfer certificates are separate purposes. Successful mTLS also does not replace Stronghold segment identity, hashing, destination verification, commit, and acknowledgement.

## Time and Clock Truthfulness

Stronghold stores authoritative wall-clock timestamps in UTC. Local timezones are presentation only.

Event ordering must not depend solely on wall-clock time. Journals and critical state maintain monotonic/advancing ordering so a wall-clock correction cannot silently reverse causality.

Initial time synchronization is expected to support multiple configured NTP sources. NTS and PTP/hardware-assisted synchronization may be added later where justified.

Stronghold distinguishes states such as:

```text
SYNCHRONIZED
HOLDOVER
UNSYNCHRONIZED
CLOCK_FAULT
```

Clock corrections, source failures, significant offset changes, and changes in clock confidence are journaled. Loss of external time synchronization must not stop capture, routing, or firewall enforcement, but affected timestamps must not be presented with false confidence.

Net-Hunter preserves the originating FW observation time separately from Hunter receipt, verification, processing, and commit time.

> **Timestamp precision must never be presented as timestamp accuracy.**

## Packet History and Journals

Raw PCAPNG remains authoritative for what network traffic Stronghold observed. The operational history around that traffic is not treated as one generic log.

Stronghold maintains separate append-oriented journal domains, including:

```text
Administrative Journal
    authentication, MFA, role use, candidate/config actions,
    commits, rollbacks, exports, break-glass, updates, reboot/shutdown

System / Health Journal
    interface/NIC state, capture loss, storage, services,
    Hunter availability, route/WAN health, ZFS/jail health

Traffic Decision Journal
    authorization, bridge/route, firewall, NAT, WAN selection,
    final disposition, configuration generation, NOT_PERFORMED states

Trust / Identity Journal
    appliance enrollment, peer authorization, certificate rotation,
    revocation, trust failures, unknown/rejected peers

Time / Clock Journal
    synchronization state, source health, offset changes,
    clock corrections, holdover and fault transitions

Hunter Processing Journal
    receipt, verification, commit, indexing, reprocessing,
    derived export creation and other Hunter processing transitions
```

These journals are logically separate even if a future storage engine shares underlying infrastructure.

Committed journal entries are append-oriented. A prior entry is not silently edited to make current state look cleaner. Corrections, superseding interpretations, recovery, retention actions, and later success after failure are represented by new entries.

Critical journals should support integrity chaining/tamper detection; the exact cryptographic journal contract remains to be frozen. PCAP integrity remains segment-oriented rather than requiring a hash chain over every individual packet.

Stronghold should explicitly preserve states such as `NOT_PERFORMED`. For example, traffic denied at the authorization gate should be able to show that routing and NAT were deliberately not performed.

Net-Hunter preserves source FW journal identity/ordering and appends its own Hunter-side journal entries; it does not rewrite the originating FW history.

## Retention, Holds, and Controlled Destruction

Stronghold applies retention independently to authoritative PCAP and to each journal domain. PCAP retention does not silently determine Administrative, System/Health, Traffic Decision, Trust/Identity, Time/Clock, or Hunter Processing Journal retention.

Retention policies may use explicit retention classes attached to configured objects or scope such as appliance, site, VLAN, zone, or other approved administrative context. Stronghold must not make destructive retention decisions depend on unproven deep-packet classification merely because a parser can derive a label.

The retention lifecycle is conceptually:

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

A legal, investigative, or administrative hold overrides ordinary expiration. Creating, changing, and releasing a hold is itself an Administrative Journal event.

Destruction must be attributable. Retention-driven and manual destruction are distinct actions, and manual destructive actions require explicit privileged authorization and a reason. The architecture leaves room for stronger controls such as MFA reauthentication or dual authorization without requiring them for every initial deployment.

Destruction does not erase the fact that history once existed. The remaining journal history should preserve the applicable object identity, original capture/time range where relevant, original integrity identity/hash where safe and required, destruction authority, reason/policy, and destruction time/order.

### FW storage pressure

Stronghold FW exposes explicit storage-pressure states such as:

```text
NORMAL
HIGH
URGENT
CRITICAL
```

Pressure may throttle/pause secondary work and accelerate safe movement/transfer, but it does not silently invent a new retention policy.

Acknowledged history that Net-Hunter has independently verified and committed is safer to expire locally than unacknowledged history. By default, Stronghold must not automatically delete unacknowledged authoritative FW history merely to hide storage pressure.

If local storage becomes exhausted and durable capture can no longer continue, Stronghold reports a real capture/history gap while continuing routing/firewall operation where possible. A full disk is a failure condition, not permission to claim continuity that did not occur.

### Net-Hunter storage pressure

Net-Hunter should expose FAST/WARM/HISTORY utilization, ingest rate, expiration rate, net growth, backlog, and projected capacity where sufficient data exists.

Hunter pressure should first preserve ingest and verification while reducing nonessential processing and advancing only already-authorized retention work. If Hunter cannot durably commit new history, it must not ACK that history; the FW retains backlog locally according to the existing handoff contract.

Moving history to a future archive is a lifecycle transition, not destruction, and must preserve identity, integrity, and lineage.

## FW storage direction

```text
RAM buffer / capture ring
    ↓
NVMe — HOT PCAP tier — XFS
    ↓
local SSD — WARM / backlog tier — XFS
    ↓
dedicated Stronghold history network
    ↓
Stronghold Net-Hunter
```

The Arch Linux operating system is expected to live on separate local SSD storage, with Btrfs as the current preferred OS-filesystem direction.

PCAP storage must remain separate from the operating-system filesystem so capture-storage exhaustion does not silently become root-filesystem exhaustion.

Net-Hunter availability must not determine whether live FW capture continues. A Hunter outage creates local backlog and an explicit degraded state while local capacity remains available.

## Dedicated History Network

Stronghold FW and Stronghold Net-Hunter communicate over a dedicated history-transfer interface or network.

Current physical link direction:

```text
10 GbE / 25 GbE / 40 GbE
```

The history-link speed is independent of the validated Stronghold FW forwarding/capture dataplane rating. A 25 GbE or 40 GbE history interface does not constitute a claim that the firewall dataplane itself has been validated at that rate.

Completed FW capture segments and associated journals/records are transferred to Net-Hunter only after required local finalization and integrity work. Net-Hunter independently verifies received history before acknowledging it as committed.

## Stronghold Net-Hunter

Stronghold Net-Hunter holds network history and provides the hunt/query system.

Current platform direction:

- FreeBSD;
- ZFS;
- jails;
- large ECC RAM capacity;
- NVMe for fast ingest/index/query workloads;
- local SAS SSD for warm storage as required; and
- HBA-attached SAS storage for large ZFS historical PCAP capacity.

The FreeBSD host owns hardware, HBAs, ZFS pools/datasets, host networking, PF, jail lifecycle, system updates, and hardware health. Application jails receive only the storage and network access required for their role.

### Net-Hunter jails

Net-Hunter currently has four defined application jails:

1. **PCAP Data Ingest Jail** — receives capture segments, source journals, and initial records from explicitly authorized Stronghold FW appliances; verifies transfer integrity/completeness; commits received history; and acknowledges verified receipt.
2. **Record Processing Jail** — reads authoritative PCAP and FW history, constructs and enriches searchable records, correlates network/firewall history, builds indexes, and may reprocess older PCAP when decoders improve.
3. **External User Interface Jail** — provides hunt, query, timeline, correlation, viewing, reporting, and controlled export. Authoritative Stronghold traffic history, journals, and records are read-only from this jail.
4. **FW Configuration Backup Jail** — receives and preserves versioned Stronghold FW configuration backups. Access is restricted to authorized Stronghold FW appliances and a local Net-Hunter administrator; ordinary hunt/UI users and the other application jails do not receive access.

### External UI read-only invariant

> **The External User Interface may query, interpret, correlate, display, and export Stronghold history, but it may not alter authoritative PCAP, source journals, observation records, processed records, indexes, or firewall configuration history.**

Writable UI storage, when required, is limited to non-authoritative material such as temporary session state, derived exports, reports, investigation notes, and download staging. User-created material becomes new non-authoritative history; it does not mutate the source record it references.

## Current Engineering Scope

The implementation roadmap still begins with the Stronghold FW capture foundation. The broader dual-appliance, networking, policy, routing, identity, trust, time, journal, retention, and configuration architecture is recorded now so early implementation choices do not block the intended complete system.

Later networking/enforcement phase sequencing is intentionally not frozen yet.

See [`docs/ROADMAP.md`](docs/ROADMAP.md) and [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Engineering Standard

Stronghold follows the Iron Signal Systems Engineering Standards pinned by the repository `ENGINEERING-STANDARD` file.

Repository and implementation behavior for contributors and coding agents is defined by `AGENTS.md` and any nested `AGENTS.md` files.

## Project Status

Stronghold is pre-release and under active development. Interfaces, schemas, storage behavior, capture implementation details, Net-Hunter internals, and architecture may change before a supported release.

## License

Stronghold is proprietary source-available software. See [`LICENSE`](LICENSE).

## Security

Please do not report suspected vulnerabilities through public GitHub issues. See [`SECURITY.md`](SECURITY.md) for the private reporting process.
