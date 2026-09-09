# Stronghold

**Stronghold by Iron Signal Systems**

Stronghold is an enforcement platform that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.

> **Observe the truth. Preserve the history. Explain the decision.**

Stronghold is one platform with explicit product boundaries:

```text
Stronghold FW
    physical observation / enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    standalone or FW-integrated access-control system

Stronghold Agent
    endpoint application / endpoint Policy Enforcement Point
```

The products cooperate through versioned, authenticated contracts. They are not one monolithic program and do not gain one another's internal authority merely because they belong to the Stronghold platform.

See [`docs/PROJECT-BOUNDARIES.md`](docs/PROJECT-BOUNDARIES.md).

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

> **A secure tunnel is a protected path, not authorization to use every resource behind it.**

> **Endpoint ALLOW never grants network permission that Stronghold FW would otherwise deny.**

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
bytes received                  != history durably committed
archived                        != destroyed
forwarding HA succeeded         != capture continuity guaranteed
virtual cluster identity        != physical capture identity
software installed              != Stronghold update validated
configuration restored          != appliance validated
backup written                  != backup proven restorable
ZFS redundancy                  != disaster recovery
disk encrypted                  != secrets safely designed
TPM protected                   != disaster recoverable
link speed                      != validated dataplane rate
capture throughput              != full-feature firewall throughput
node performance                != HA cluster performance
storage capacity                != sustainable ingest capacity
AF_XDP available                != AF_XDP zero-copy available
zero-copy available             != Stronghold zero-copy qualified
WireGuard peer authenticated    != user authenticated
tunnel established              != resource authorized
endpoint ALLOW                  != FW ALLOW
endpoint DENY                   != FW observed DENY
device on VLAN                  != device authorized for VLAN/zone
policy received                 != policy activated
policy activated                != endpoint enforcement healthy
```

## Stronghold FW

Stronghold FW is the live-network appliance.

Current platform direction:

```text
Arch Linux / x86_64
CLI-first appliance
qualified physical PCIe Ethernet NICs
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

### Capture Requirement — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline.

Stronghold FW uses **AF_XDP as the intended packet-acquisition foundation**. Queue/RSS behavior, UMEM/rings, copy/native/zero-copy state, NIC/driver/firmware behavior, drop accounting, CPU affinity, NUMA placement, PCIe topology, packet size, and PPS are qualified rather than assumed.

See [`docs/AF-XDP.md`](docs/AF-XDP.md).

### Enforcement Model

Stronghold supports Layer-2 transparent bridging, Layer-3 IPv4/IPv6 routing, router-on-a-stick over 802.1Q trunks, and hybrid deployments.

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

A route lookup never grants authorization. NAT never grants permission. Multi-WAN selects only among paths explicitly eligible for the destination/service.

### Standalone FW and Local Working History

Stronghold FW is intended to remain a useful firewall without Net-Hunter.

The local history model serves two purposes:

```text
NORMAL OPERATOR WORKING SET
    approximately 30–60 minutes
    near-real-time policy / route / NAT / WAN validation

HUNTER OUTAGE BUFFER
    approximately 4 hours without Hunter is CRITICAL
    qualified local capacity direction is roughly 8 hours equivalent ingest
```

Net-Hunter is not required for an administrator to determine what the FW is doing now. Recent traffic and decisions should become locally visible in seconds, not wait for long-term Hunter indexing.

Backlog catch-up is subordinate to live observation. Recovering an old transfer backlog must never starve current packet acquisition or active PCAP writes.

## Journals and External Log Integration

Stronghold maintains separate append-oriented operational journal domains rather than one mutable catch-all log.

Authoritative journal state remains Stronghold-owned. Structured copies may be exported independently to Net-Hunter, Security Onion, Graylog, or other approved SIEM/log-ingestion systems.

```text
Stronghold FW Journals
        │
        ├── local recent operational view
        ├── Net-Hunter verified history
        ├── Security Onion / SIEM
        ├── Graylog / log ingestion
        └── other approved receivers
```

```text
journal committed locally != SIEM event delivered
SIEM event delivered      != Net-Hunter verified history committed
not exported              != not journaled
SIEM export gap           != Stronghold journal gap
```

See [`docs/JOURNAL-EXPORT.md`](docs/JOURNAL-EXPORT.md).

## Stronghold Net-Hunter

Net-Hunter runs on FreeBSD with ZFS and jails and owns long-term historical responsibilities:

```text
receive / verify / durably commit
preserve authoritative PCAP and source journals
process / reprocess
correlate
hunt / query
export
retention / holds / archive
FW configuration backup
future isolated IDS/inspection analysis
```

Net-Hunter is optional for basic FW operation, but it is what turns Stronghold from a short-window enforcement appliance into a durable historical and forensic platform.

Defined application jails include:

1. **PCAP Data Ingest Jail**
2. **Record Processing Jail**
3. **External User Interface Jail**
4. **FW Configuration Backup Jail**
5. **IDS / Inspection Jail** — future/optional

The External UI remains read-only with respect to authoritative history. Derived indexes and correlations may be rebuilt without rewriting authoritative PCAP or source journals.

See [`docs/NET-HUNTER-RECORDS.md`](docs/NET-HUNTER-RECORDS.md).

## Stronghold Access

Stronghold Access is a separate access-control product that may run on a qualified customer VM or bare-metal host. It can operate standalone or integrate with Stronghold FW through a narrow authenticated control contract.

Stronghold Access owns:

```text
identity inputs
device trust / posture
first-class 802.1X network admission
AAA integration
Network Access Sessions
Stronghold Access Sessions
resource-scoped authorization
revocation / reevaluation
policy distribution to Stronghold Agent
controlled Stronghold FW integration
```

Remote zero-trust access follows the NIST SP 800-207 PE/PA/PEP direction with WireGuard as the secure transport/data plane. WireGuard does not become the authorization system.

Site-to-site / branch-office tunneling is a separate architecture using L2TPv3 protected by IPsec and normal Stronghold policy around the tunnel.

See [`docs/access/ARCHITECTURE.md`](docs/access/ARCHITECTURE.md) and [`docs/SECURE-ACCESS.md`](docs/SECURE-ACCESS.md).

## Stronghold Agent

Stronghold Agent is a separate endpoint product and endpoint Policy Enforcement Point. Windows is the initial platform direction, with implementation reserved under `go/agent/`.

Initial endpoint enforcement direction:

```text
Stronghold Access / authenticated network context
        ↓
signed monotonic endpoint policy generation
        ↓
Stronghold Agent
    Windows service / Go
        ↓
Windows Filtering Platform
        ↓
process/application-aware local enforcement
        ↓
normal network path or WireGuard
        ↓
Stronghold FW
        ↓
independent network authorization
```

The Agent may reject unauthorized process/application connections locally before they consume network or FW resources.

```text
endpoint ALLOW != FW ALLOW
endpoint DENY  != FW observed DENY
```

Network-side VLAN/zone context is stronger than endpoint self-assertion. An Agent cannot declare itself trusted because it reports a particular VLAN or network.

The initial Agent direction is application/process-aware connection enforcement, not general endpoint deep-payload inspection. Any future local proxy or kernel-mode WFP callout-driver design is separately qualified.

Protected Endpoint mode may later force eligible traffic from selected high-value systems through an authenticated encrypted Stronghold transport while still preserving necessary local-link traffic and truthful metadata visibility.

See [`docs/agent/ARCHITECTURE.md`](docs/agent/ARCHITECTURE.md).

## 802.1X and Multiple Enforcement Points

802.1X is a first-class Stronghold Access security primitive, not an afterthought.

A mature deployment may include:

```text
NETWORK-ADMISSION PEP
    switch / AP / 802.1X

ENDPOINT PEP
    Stronghold Agent / WFP

NETWORK PEP
    Stronghold FW
```

A decision at one enforcement location does not fabricate a decision at another.

## Secure Access

Stronghold separates two future secure-access families:

```text
ZERO-TRUST REMOTE ACCESS
    NIST SP 800-207 control model
    Stronghold Access PE / PA
    Stronghold Agent endpoint PEP
    Stronghold FW network PEP
    WireGuard secure transport
    user + device + MFA + posture + resource authorization
    first-class GRANT / DENY / REVOKE

SITE-TO-SITE / BRANCH OFFICE
    L2TPv3
    IPsec protection
    site/tunnel identity
    VLAN / bridge / routing integration
    normal Stronghold policy enforcement
```

Secure-access implementation remains deferred even though the architecture is defined.

## IDS / TLS Inspection

Future selective TLS/application inspection is also architecture-defined but implementation-deferred.

The FW retains authoritative encrypted-wire PCAP. Where inspection is explicitly selected, plaintext exists only in controlled volatile processing on the FW and is transported over a separate authenticated encrypted INSPECTION path to a future isolated Hunter IDS/Inspection Jail.

Stronghold FW never intentionally persists decrypted inspection payload at rest.

See [`docs/IDS-INSPECTION.md`](docs/IDS-INSPECTION.md).

## Configuration, Simulation, and Management

Stronghold uses one configuration authority with transactional generations:

```text
RUNNING
  ↓
CANDIDATE
  ↓
VALIDATE
  ↓
SHOW DIFF
  ↓
SIMULATE where applicable
  ↓
COMMIT
  ↓
NEW GENERATION
```

CLI, API, and future UI are clients of the same configuration and authorization machinery. Native OS state is implementation state below Stronghold, not a second configuration truth.

Policy simulation/counterfactual testing is intended to show expected policy, route, NAT, WAN, and session impact before activation.

See [`docs/CONFIGURATION-GOVERNANCE.md`](docs/CONFIGURATION-GOVERNANCE.md), [`docs/POLICY-SIMULATION.md`](docs/POLICY-SIMULATION.md), and [`docs/MANAGEMENT-PLANE.md`](docs/MANAGEMENT-PLANE.md).

## High Availability

Initial FW HA is optional **active/standby** only.

Each node retains its own Appliance ID, physical capture provenance, local storage, certificates, clock state, and node-local configuration. The cluster owns the virtual forwarding identity.

> **Virtual network identity is forwarding identity, not capture identity.**

Loss of peer communication is not, by itself, proof the peer is dead. Promotion requires a later explicit fencing/election contract. Net-Hunter is not an HA witness/quorum dependency.

## Recovery, Encryption, and Platform Trust

Stronghold separates rebuildable platform state, recoverable configuration, appliance/trust identity, authoritative history, derived state, and transient runtime state.

FW storage direction uses encrypted block devices beneath Btrfs/XFS. Hunter favors purpose-separated ZFS-native encrypted datasets. TPM/HSM mechanisms may protect normal key use but must not become the sole recovery path for authoritative history.

Secure Boot, measured boot, and attestation remain separate facts. Development hardware, qualification hardware, and supported production hardware remain distinct categories.

## Resource Priority

On Stronghold FW:

```text
1. AF_XDP packet acquisition / ring service
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation / decision / journal append
5. HA heartbeat/control reserved capacity
6. journal checkpoint/finalization
7. local tier movement / backlog
8. HA bulk state synchronization
9. authoritative Net-Hunter HISTORY transfer
10. compression where approved
11. deeper analytics / observability / support / IDS / access housekeeping
```

Secondary work may degrade or lag. It must not silently win over authoritative live observation.

## Current Implementation Scope

Implementation begins with **Phase 0 — Traffic Observation Foundation** using AF_XDP.

Phase 0 proves:

```text
capture segment contract
traffic / segment catalog contract
PCAPNG metadata contract
local storage contract
capture resource protection
single-interface AF_XDP capture
multi-interface / multi-queue AF_XDP capture
PPS / loss / visibility / hardware qualification baseline
local tier movement
verified Net-Hunter history transfer
```

Secure Access, Stronghold Agent endpoint enforcement, IDS/IPS, HA, full firewall enforcement, and other later systems remain outside Phase 0 even where their architecture is already defined.

See [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Engineering Standard

Stronghold follows the Iron Signal Systems Engineering Standards pinned by `ENGINEERING-STANDARD`. Contributor behavior is governed by `AGENTS.md` and nested `AGENTS.md` files.

## Project Status

Stronghold is pre-release and under active development. Architecture and interfaces may change before a supported release.

## License

Stronghold is proprietary source-available software. See [`LICENSE`](LICENSE).

## Security

Do not report suspected vulnerabilities through public GitHub issues. See [`SECURITY.md`](SECURITY.md).
