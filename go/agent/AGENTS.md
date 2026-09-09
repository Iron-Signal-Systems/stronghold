# Stronghold Agent Go Project Rules

## Scope

This file governs implementation under:

```text
go/agent/
```

This subtree is reserved for **Stronghold Agent**, the endpoint application and endpoint Policy Enforcement Point.

Stronghold Agent is a separate project from:

```text
go/access/
Stronghold FW implementation
Stronghold Net-Hunter implementation
```

Do not place Stronghold Access server/control-plane implementation, Stronghold FW dataplane/capture code, or Net-Hunter processing code in this subtree.

See `docs/agent/ARCHITECTURE.md`.

## Initial Platform

Windows is the first Agent platform direction.

Windows-owned implementation should use appropriate native Windows facilities and APIs. Endpoint enforcement is expected to use Windows Filtering Platform where the frozen implementation design requires it.

Do not implement Windows behavior by repeatedly parsing PowerShell, `netsh`, WMI CLI output, or other shell text when an appropriate native API exists.

Future Linux or macOS Agent implementations, if approved, must target those platforms natively rather than forcing Windows-specific abstractions across all operating systems.

## Product Responsibility

Stronghold Agent may own endpoint responsibilities such as:

```text
endpoint identity presentation
endpoint enrichment
user / subject context
application / process identity
posture collection approved by architecture
local endpoint policy enforcement
Protected Endpoint secure transport
Access Session participation
endpoint health
endpoint decision records
revocation handling
```

Stronghold Agent does not own the Stronghold Access Policy Engine, Stronghold FW network authorization, network-admission authority, or Net-Hunter historical authority.

## Stronghold Access Boundary

Stronghold Access implementation lives under:

```text
go/access/
```

Agent code must not import Access internal implementation packages.

The Agent receives authenticated/versioned policy and session state through explicit contracts.

Preserve:

```text
policy received  != policy verified
policy verified  != policy activated
policy activated != endpoint enforcement healthy
```

The Agent is not an independent Stronghold Policy Engine.

## Stronghold FW Boundary

Stronghold FW remains an independent network PEP.

Preserve:

```text
endpoint ALLOW != FW ALLOW
endpoint DENY  != FW observed DENY
```

If the Agent denies a connection before network transmission, record an endpoint denial. Do not manufacture a FW observation for traffic the FW never received.

## Process / Application Identity

Endpoint-origin process identity is different from network traffic interpretation.

Preserve:

```text
Windows originating process
!=
network application classification
```

Process identity may include, when explicitly supported and established:

```text
executable name
process path
publisher/signature state
hash
user / token context
requested destination/resource
service/protocol
```

Do not claim stronger identity than Windows and the selected collection/enforcement point can actually establish.

## Protected Endpoint

Protected Endpoint is an explicit operating profile for selected high-value endpoints.

The implementation must not silently call ordinary default-route VPN behavior a Stronghold full-tunnel guarantee.

Qualification must account for bypass/leak surfaces including:

```text
IPv4
IPv6
DNS
local-subnet routing
secondary NICs
Wi-Fi + Ethernet concurrency
USB NICs
virtual adapters
Hyper-V networking
other VPN software
boot/startup
sleep/resume
network transitions
route changes
multicast/broadcast
DHCP/DHCPv6
ARP/NDP
Agent failure
transport failure
local-administrator tampering
```

Local-link traffic required to establish and maintain network connectivity must be explicitly identified rather than hidden.

Protected Endpoint may prevent plaintext application conversations from traversing an otherwise untrusted local network, but it does not make the endpoint or encrypted traffic metadata invisible.

## WireGuard Boundary

WireGuard is the current Stronghold protected remote-access transport direction.

Preserve:

```text
WireGuard peer authenticated != user authenticated
tunnel established           != resource authorized
process authorized           != FW resource authorized
```

The Agent must not convert transport availability into authorization.

## 802.1X Boundary

Stronghold Agent does not replace the native Windows 802.1X supplicant or enterprise NAC infrastructure without a separately approved requirement.

802.1X/network admission is modeled by Stronghold Access.

Agent-provided network context is not allowed to self-assert trusted VLAN/zone/admission state.

```text
Agent reports VLAN/zone != network-side VLAN/zone established
```

## Shared-Code Rule

Do not create generic packages named `common`, `shared`, `util`, `helpers`, or `framework` merely to share implementation with Access or FW.

If a real shared wire/schema contract is required, freeze and version that contract explicitly. Keep endpoint-specific behavior in the Agent project.

## Go Engineering

All parent `go/AGENTS.md` requirements apply unless this file narrows them for Stronghold Agent.

Potential future package directions, only when implementation requires them, may include:

```text
internal/control/
internal/enforcement/
internal/identity/
internal/policy/
internal/posture/
internal/process/
internal/transport/
internal/windows/
```

Do not create packages before roadmap work requires them.

Keep Windows-native and unsafe/native-boundary code narrow, validated, and isolated.

## Endpoint Truth Discipline

Examples:

```text
Agent service running        != endpoint PEP healthy
process path observed        != publisher trusted
certificate valid            != device authorized
posture unavailable          != posture passed
tunnel up                    != resource authorized
endpoint policy current      != FW policy current
endpoint denied locally      != FW denied packet
```

## Security Principle

> **Stronghold Agent establishes and enforces the endpoint side of the access decision. It does not impersonate Stronghold Access or Stronghold FW.**
