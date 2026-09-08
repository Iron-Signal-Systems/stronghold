# Stronghold

**Stronghold by Iron Signal Systems**

Stronghold is a two-appliance network security system built around complete network observation, comprehensive records, policy enforcement, and retained packet history.

**Stronghold FW** runs on Arch Linux and observes, records, bridges or routes, and enforces live traffic.

**Stronghold Net-Hunter** runs on FreeBSD with ZFS and jails and receives verified packet history, processes network records, preserves historical PCAP, and provides a read-only hunt/query interface.

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

## Governing Principle

> **Capture first. Never sacrifice observation for secondary work.**

Live packet capture and durable local writes take priority over compression, transfer, indexing, analytics, and other background work.

## Capture Requirement #1 — Wireshark-Class Interface Visibility

Stronghold's first capture requirement is simple:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

The intended visibility baseline is Wireshark/dumpcap on the same supported physical interface under the same conditions.

Protocol recognition is not required before capture. When presented to the interface, Stronghold should preserve ordinary IPv4/IPv6 traffic as well as Layer-2/control-plane traffic such as ARP, DHCP, DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes, unknown IP protocols, vendor-specific frames, and malformed traffic.

Stronghold does not claim visibility into traffic that the network topology, NIC hardware, hardware filtering/offload behavior, or driver never presents to the supported capture path.

## Stronghold FW

Stronghold FW is the live network appliance.

Current direction:

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
- hybrid deployments in which different interfaces, VLANs, or bridge domains use different forwarding models;
- nftables stateful firewalling, NAT, and policy enforcement as later layers built on the capture foundation.

### FW networking model

Stronghold does not require the entire firewall appliance to operate in one global forwarding mode.

Supported architectural deployment models include:

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

VLANs are first-class Stronghold objects rather than anonymous numeric values embedded only in Linux interface names. Stronghold records preserve both the human Stronghold object identity and the actual VLAN ID.

The broader object model is expected to include physical interfaces, logical interfaces, VLANs, bridge domains, zones, hosts, networks, address groups, services, service groups, FQDNs, and FQDN groups. Exact schemas remain to be designed.

FQDN policy is expected to distinguish DNS-resolved address-set policy from future true Layer-7 hostname observation. Stronghold must not treat an IP obtained from DNS as proof that a specific connection actually carried that hostname.

### FW storage direction

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

Completed FW capture segments and their associated records are transferred to Net-Hunter only after the required local finalization and integrity work. Net-Hunter independently verifies received history before acknowledging it as committed.

## Records

Raw PCAPNG is the authoritative packet history. Structured records make that history searchable, explainable, and correlatable.

Stronghold FW should create the initial records that can be established while traffic is live, including physical-interface observation, ingress/egress context, timestamps, MAC/VLAN context, bridge-domain context, flow/session facts, protocol/control-plane facts when safely decoded, Layer-2 forwarding decisions, Layer-3 routing decisions, firewall decisions, NAT decisions, packet-loss state, and system/failure activity.

Stronghold must preserve the distinction between physical observation and forwarding disposition. For example, router-on-a-stick traffic may enter and leave the same physical trunk with different VLAN context; the records must preserve that relationship without pretending the observations are unrelated traffic.

Stronghold Net-Hunter may later enrich, correlate, and reprocess historical PCAP with improved decoders.

Failure to recognize or decode a protocol must never cause the underlying packet to be discarded.

## Stronghold Net-Hunter

Stronghold Net-Hunter holds network history and provides the hunt/query system.

Current platform direction:

- FreeBSD;
- ZFS;
- jails;
- large ECC RAM capacity;
- NVMe for fast ingest/index/query workloads;
- local SAS SSD for warm storage as required;
- HBA-attached SAS storage for large ZFS historical PCAP capacity.

The FreeBSD host owns hardware, HBAs, ZFS pools/datasets, host networking, PF, jail lifecycle, system updates, and hardware health. Application jails receive only the storage and network access required for their role.

### Net-Hunter jails

Net-Hunter currently has four defined application jails:

1. **PCAP Data Ingest Jail** — receives capture segments and initial records from authorized Stronghold FW appliances, verifies transfer integrity/completeness, commits received history, and acknowledges verified receipt.
2. **Record Processing Jail** — reads authoritative PCAP and FW records, constructs and enriches searchable records, correlates network/firewall history, builds indexes, and may reprocess older PCAP when decoders improve.
3. **External User Interface Jail** — provides hunt, query, timeline, correlation, viewing, reporting, and controlled export. Authoritative Stronghold traffic history and records are read-only from this jail.
4. **FW Configuration Backup Jail** — receives and preserves versioned Stronghold FW configuration backups. Access is restricted to authorized Stronghold FW appliances and a local Net-Hunter administrator; ordinary hunt/UI users and the other application jails do not receive access.

### External UI read-only invariant

> **The External User Interface may query, interpret, correlate, display, and export Stronghold history, but it may not alter authoritative PCAP, observation records, processed records, indexes, or firewall configuration history.**

Writable UI storage, when required, is limited to non-authoritative material such as temporary session state, derived exports, reports, and download staging.

## Current Engineering Scope

The present implementation roadmap still begins with the Stronghold FW capture foundation. The broader dual-appliance and networking architecture is recorded now so early capture, record, storage, interface, VLAN, and transfer decisions do not block the intended complete system.

Layer-2 bridging, routing, VLAN configuration, subinterfaces, nftables policy enforcement, FQDN policy objects, Net-Hunter processing, and UI implementation remain later engineering work until explicitly brought into scope.

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
