# Security Policy

## Reporting a Security Issue

Please do **not** report suspected vulnerabilities through public GitHub issues.

If you believe you have found a security issue in Stronghold, report it privately to the project maintainer.

Include, where possible:

- a clear description of the issue;
- affected component or file;
- steps to reproduce;
- expected and observed behavior;
- potential security impact; and
- any supporting logs, traces, or proof-of-concept details.

Please avoid including real customer data, credentials, secrets, packet captures containing sensitive production data, or other sensitive information.

## Project Status

Stronghold is currently pre-release and under active development.

Security behavior, interfaces, schemas, storage behavior, capture implementation details, routing behavior, and policy behavior may change before a supported release.

## Disclosure

Please allow reasonable time for investigation and remediation before publicly disclosing a reported vulnerability.

Validated security issues will be addressed according to their severity and impact.

## Scope

Security reports relating to Stronghold-controlled components are welcome, including:

- Arch Linux appliance configuration;
- packet capture and AF_PACKET / TPACKET_V3 handling;
- capture buffers and packet-loss accounting;
- PCAP/PCAPNG generation;
- capture integrity and hashing;
- capture-segment and traffic catalogs;
- storage tiering, retention, and deletion;
- compression and resource-priority behavior;
- SMB, SFTP, SSH, and iSCSI-related offload handling;
- credentials, keys, and secret handling;
- CLI and future management interfaces;
- routing, VLAN, and subinterface handling when implemented;
- nftables/firewall enforcement when implemented; and
- FQDN policy processing when implemented.

Issues in third-party products or infrastructure that Stronghold does not control should normally be reported to the applicable vendor.

## Contact

Security reports should be sent privately to the project maintainer.

**Email:** security@ironsignalsystems.com  
**Maintainer:** John Joseph Wood  
**Project:** Stronghold / Iron Signal Systems
