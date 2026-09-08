# Stronghold

**Stronghold by Iron Signal Systems**

Stronghold is a capture-first network security appliance project. The initial platform is a stripped-down Arch Linux x86_64 system with a simple CLI-focused operating model.

Stronghold begins with observation before enforcement: continuously capture traffic, catalog what was observed, preserve packet-level truth, tier completed captures to appropriate storage, and make those captures queryable and exportable. Routing, VLAN/subinterface configuration, and firewall policy enforcement are later layers built on top of that observation foundation.

## Governing Principle

> **Capture first. Never sacrifice observation for secondary work.**

Live packet capture and durable local writes take priority over compression, tier migration, remote offload, indexing, analytics, and other background work.

## Capture Requirement #1 — Wireshark-Class Interface Visibility

Stronghold's first capture requirement is simple:

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, routes, or firewall-processes it.**

The intended visibility baseline is Wireshark/dumpcap on the same supported physical interface under the same conditions.

Protocol recognition is not required before capture. When presented to the interface, Stronghold should preserve ordinary IPv4/IPv6 traffic as well as Layer-2/control-plane traffic such as ARP, DHCP, DHCPv6, CDP, LLDP, STP/RSTP/MSTP, LACP, 802.1X/EAPOL, OSPF, VRRP, IGMP, IPv6 NDP, VLAN-tagged traffic, unknown EtherTypes, unknown IP protocols, vendor-specific frames, and malformed traffic.

Stronghold does not claim visibility into traffic that the network topology, NIC hardware, hardware filtering/offload behavior, or driver never presents to the supported capture path.

## Initial Platform

- Arch Linux, minimal CLI installation
- x86_64
- physical PCIe Ethernet interfaces with strong Linux driver support
- Linux-native packet capture
- AF_PACKET / TPACKET_V3 as the initial capture direction
- continuous full-packet PCAPNG capture
- local-first durable storage
- CLI administration

## Capture Storage Model

Stronghold uses downward storage tiering:

```text
NIC
 ↓
RAM buffer / capture ring
 ↓
NVMe — hot ingest tier
 ↓
SSD — warm tier
 ↓
HDD / RAID — cold tier
 ↓
optional selected offload/archive
```

RAM is buffering only and is not considered durable capture storage.

Completed segments may be compressed while tiering downward when CPU and I/O resources allow. Compression must automatically throttle or pause when capture resources are under pressure.

## Offload Model

Stronghold captures the configured interfaces locally first. Offload policy is separate from capture policy.

For example, a system may capture all traffic observed on `eth0` and `eth1` while retaining or offloading selected VLANs such as VLAN 1, VLAN 2, and VLAN 201 to approved destinations.

Planned destination classes include SMB/UNC-backed storage, SFTP/SSH-based transfer, and iSCSI-backed mounted storage.

Remote storage availability must not determine whether local capture continues.

## Current Scope

The current engineering scope is the capture foundation only:

- Wireshark-class physical-interface visibility
- packet acquisition
- packet-loss accounting
- PCAPNG segment creation
- segment finalization and integrity
- capture/flow cataloging
- storage tiering
- resource-aware compression
- retention
- selective VLAN offload
- capture query and export

Routing, VLAN configuration, subinterfaces, nftables policy enforcement, and FQDN policy objects are intentionally later phases.

See [`docs/ROADMAP.md`](docs/ROADMAP.md) and [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Engineering Standard

Stronghold follows the Iron Signal Systems Engineering Standards pinned by the repository `ENGINEERING-STANDARD` file.

Repository and implementation behavior for contributors and coding agents is defined by `AGENTS.md` and any nested `AGENTS.md` files.

## Project Status

Stronghold is pre-release and under active development. Interfaces, schemas, storage behavior, capture implementation details, and architecture may change before a supported release.

## License

Stronghold is proprietary source-available software. See [`LICENSE`](LICENSE).

## Security

Please do not report suspected vulnerabilities through public GitHub issues. See [`SECURITY.md`](SECURITY.md) for the private reporting process.
