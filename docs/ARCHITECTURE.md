# Stronghold Architecture

## Purpose

This document records the integrated high-level architecture of the Stronghold platform. Focused subsystem documents govern their respective implementation details.

> **Stronghold is an enforcement system that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.**

Stronghold is one platform with explicit product and implementation boundaries:

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

The product split is governed by `docs/PROJECT-BOUNDARIES.md`. Cooperation does not imply shared internal authority or one monolithic implementation.

```text
OBSERVE → RECORD → ENFORCE → REMEMBER → HUNT
```

> **Observe the truth. Preserve the history. Explain the decision.**

## Truth Model

```text
WHAT WAS PRESENTED
    authoritative physical-interface observation / PCAPNG

WHAT STRONGHOLD DID
    authoritative operational and decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation, correlation, and enrichment
```

### Observation survives interpretation

If Stronghold preserves a frame that it cannot decode today, that packet remains authoritative history. Later decoders or correlation may create new derived interpretation without rewriting the source observation.

Missing interpretation must never erase observation, and missing history must never be presented as proof of absence.

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **A secure tunnel is a protected path, not authorization to use every resource behind it.**

> **Endpoint ALLOW never grants network permission that Stronghold FW would otherwise deny.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

> **Stronghold owns the supported appliance operating-system and release lifecycle.**

> **Record what the system can actually establish, preserve the source of that truth, and never manufacture certainty beyond it.**

## Common Truth Boundaries

```text
route available                    != route authorized
certificate valid                  != peer authorized
packet not decoded                 != packet not observed
no journal record                  != event did not occur
timestamp precision                != timestamp accuracy
policy allowed                     != forwarding succeeded
object expired                     != destruction authorized
bytes received                     != history durably committed
archived                           != destroyed
new interpretation                 != new historical observation
forwarding HA succeeded            != capture continuity guaranteed
virtual cluster identity           != physical capture identity
software installed                 != Stronghold update validated
configuration restored             != appliance validated
backup written                     != backup proven restorable
ZFS redundancy                     != disaster recovery
disk encrypted                     != secrets safely designed
TPM protected                      != disaster recoverable
entry written                      != journal entry durably committed
signed journal checkpoint          != physically immutable storage
system booted                      != approved system state
hardware detected                  != hardware supported
link speed                         != validated dataplane rate
capture throughput                 != full-feature firewall throughput
storage capacity                   != sustainable ingest capacity
AF_XDP available                   != AF_XDP zero-copy available
AF_XDP zero-copy available         != Stronghold zero-copy qualified
WireGuard peer authenticated       != user authenticated
tunnel established                != resource authorized
802.1X authenticated               != resource authorized
RADIUS Access-Accept               != Stronghold Access GRANT
endpoint ALLOW                     != FW ALLOW
endpoint DENY                      != FW observed DENY
device on VLAN                     != device authorized for VLAN/zone
policy received                    != policy activated
policy activated                   != endpoint enforcement healthy
L2TPv3 tunnel established          != site authorized
IPsec SA established               != traffic authorized
```

# Stronghold FW

## Platform Direction

```text
Platform:          Arch Linux
Architecture:      x86_64
Administration:    CLI first
NICs:              qualified physical PCIe Ethernet
Packet acquisition: AF_XDP
Capture format:    PCAPNG
Enforcement:       nftables/netfilter direction
OS storage:        dedicated SSD / Btrfs direction
HOT capture:       NVMe / XFS direction
WARM/backlog:      SSD / XFS direction
1 GbE:             current qualification target
10 GbE:            primary production target
40 GbE:            future target only
```

`docs/AF-XDP.md` governs the packet-acquisition architecture.

## Capture Invariant — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under equivalent supported conditions remains the reference visibility baseline.

AF_XDP is subordinate to that invariant. Queue/RSS behavior, UMEM/rings, copy/native/zero-copy mode, offload representation, timestamps, drops, CPU affinity, PCIe locality, and NUMA placement are qualification facts rather than assumptions.

## Networking and Enforcement

Stronghold supports the architecture for:

```text
Layer-2 transparent bridging
Layer-3 IPv4/IPv6 routing
router-on-a-stick over 802.1Q trunks
hybrid Layer-2/Layer-3 deployments
NAT
policy-defined multi-WAN
```

VLANs, bridge domains, zones, interfaces, routes, security policy, NAT policy, WAN eligibility, and related objects are explicit Stronghold configuration rather than hidden native state.

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

A route lookup may establish facts but never grants permission. NAT never grants permission. WAN availability never grants permission.

## Standalone FW and Local Working History

Stronghold FW is intended to remain operational and useful without Net-Hunter.

The local storage/history model has two jobs:

```text
OPERATOR WORKING SET
    approximately 30–60 minutes
    near-real-time recent traffic and decision visibility

HUNTER OUTAGE BUFFER
    approximately 4 hours without Hunter is CRITICAL
    qualified physical capacity direction: roughly 8 hours equivalent ingest
```

The administrator should be able to validate policy, routing, NAT, WAN selection, and actual forwarding outcome locally without waiting for Hunter ingestion or indexing.

Backlog recovery is subordinate to live observation. Catch-up must not starve current packet acquisition or active PCAP writes.

# Stronghold Journals and External Export

Stronghold maintains separate append-oriented journal domains:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Source journals are Stronghold authority. External SIEM/log systems receive copies/projections and do not replace the authoritative source.

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

`docs/JOURNAL-EXPORT.md` governs journal export.

# Stronghold Net-Hunter

Net-Hunter is the historical system for long-term Stronghold packet and operational history.

```text
Stronghold FW
    finalized authoritative history
        │
        │ dedicated HISTORY transport
        │ mTLS + explicit peer authorization
        ▼
Net-Hunter
    receive
    verify
    durably commit
    preserve
    process / reprocess
    correlate
    hunt / query
    export
```

Net-Hunter is optional for basic FW operation. It is what provides long-duration packet history, deep historical search, reprocessing, configuration backup, and forensic reconstruction.

Defined application jails:

1. **PCAP Data Ingest Jail** — authenticated receive, verification, durable commit, ACK.
2. **Record Processing Jail** — derives/rebuilds searchable records from authoritative history.
3. **External User Interface Jail** — read-only investigation of authoritative history with non-authoritative workspace.
4. **FW Configuration Backup Jail** — isolated versioned FW configuration history.
5. **IDS / Inspection Jail** — future/optional isolated deep-inspection processing.

The FreeBSD host owns ZFS, raw storage/HBA authority, host networking, jail lifecycle, updates, and encryption-key authority.

Derived records/indexes are rebuildable. They must never become the sole proof that authoritative traffic existed.

`docs/NET-HUNTER-RECORDS.md` governs Hunter records/search/reprocessing.

# Stronghold Access

Stronghold Access is the access-control/control-framework product. It may be deployed on a qualified customer VM or bare-metal system and may operate without Stronghold FW.

It owns access-control responsibilities including:

```text
identity inputs
device trust / posture
802.1X network admission
AAA integration
Network Access Sessions
Stronghold Access Sessions
resource-scoped authorization
revocation / reevaluation
Agent policy/session distribution
controlled FW integration
```

Stronghold Access does not own FW packet capture, routing, NAT, physical observation, or FW Traffic Decision Journal authority.

`docs/access/ARCHITECTURE.md` is the authoritative Access architecture.

## 802.1X as a First-Class Control

802.1X is a first-class access-control primitive, not an afterthought.

Stronghold preserves:

```text
network admission
!=
resource authorization
```

EAP-TLS is an important high-assurance direction where supported. MAB may be necessary for legacy/non-supplicant devices but is not represented as equivalent assurance to authenticated 802.1X.

External AAA such as NPS, ISE, ClearPass, FreeRADIUS, or other qualified RADIUS systems may provide identity/authorization attributes, but those attributes remain inputs to Stronghold Access policy rather than hidden authorization bypasses.

## Network Context Distribution

When FW and Access are integrated, the FW may provide network-side context that the endpoint cannot authoritatively self-assert.

```text
Stronghold FW / network admission
    establishes VLAN / zone / attachment facts
        │
        ▼
Stronghold Access
    correlates endpoint + network context
    evaluates policy
        │
        │ signed monotonic policy generation
        ▼
Stronghold Agent
    verifies / validates / programs WFP
```

The control path is generation-based and authenticated. It is not a per-packet command stream from FW to Agent.

# Stronghold Agent

Stronghold Agent is the endpoint application and endpoint Policy Enforcement Point. Windows is the initial platform direction.

```text
Stronghold Agent
    Windows service
    Go
        │
        ▼
Windows-native integration
        │
        ▼
Windows Filtering Platform
        │
        ▼
local endpoint PEP
```

The Agent can reject unauthorized process/application connections locally before they consume LAN, WireGuard, or FW resources.

```text
process attempts connection
        │
        ▼
Stronghold Agent / WFP
    ├── DENY → local endpoint record
    └── ALLOW
            │
            ▼
       network/tunnel
            │
            ▼
       Stronghold FW
            │
            ▼
       independent policy
```

Endpoint ALLOW cannot force FW ALLOW. Endpoint DENY is not represented as a FW denial if traffic never reached the FW.

`docs/agent/ARCHITECTURE.md` is the authoritative Agent architecture.

## Endpoint Layer-7 Boundary

The initial Windows Agent uses application/process identity as an authorization fact. That is not the same thing as general deep-payload inspection.

```text
originating process identified
!=
payload decoded

application-aware connection policy
!=
deep Layer-7 payload inspection
```

A future local proxy or kernel-mode WFP callout-driver design requires separate qualification for protocol coverage, TLS behavior, driver signing, kernel attack surface, crash/BSOD risk, upgrade compatibility, performance, privacy, retention, and failure behavior.

# Secure Access

Secure access is architecture-defined but implementation-deferred.

## Zero-Trust Remote Access

The control architecture follows the NIST SP 800-207 Policy Engine / Policy Administrator / Policy Enforcement Point direction.

```text
Stronghold Access
    Policy Engine / Policy Administrator
        │
        ├── signed/session policy → Stronghold Agent endpoint PEP
        │
        └── authenticated control → Stronghold FW network PEP

Stronghold Agent
    WireGuard secure transport
        │
        ▼
Stronghold FW
    independent resource authorization
```

WireGuard is the secure remote-access transport/data plane. It is not the authorization database.

```text
WireGuard peer authenticated != user authenticated
user authenticated           != device authorized
device authorized             != resource authorized
tunnel established            != resource authorized
```

## Site-to-Site / Branch Office

Site-to-site / branch-office tunneling is a separate architecture:

```text
L2TPv3 tunnel
    ↓
IPsec protection
    ↓
site / tunnel identity
    ↓
bridge / VLAN / routing integration
    ↓
normal Stronghold policy
```

Tunnel establishment and IPsec SA state never independently grant traffic authorization.

`docs/SECURE-ACCESS.md` retains the earlier detailed secure-access model; where product ownership differs, `docs/PROJECT-BOUNDARIES.md`, `docs/access/ARCHITECTURE.md`, and `docs/agent/ARCHITECTURE.md` take precedence.

# Configuration and Management

Stronghold management surfaces are clients of one configuration authority.

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
NEW MONOTONIC GENERATION
```

CLI, API, and future UI do not maintain separate configuration truth. Native Linux/FreeBSD/WireGuard/L2TPv3/IPsec/WFP state remains implementation state below Stronghold and must not become a hidden second configuration authority.

Policy simulation/counterfactual testing is intended to let administrators evaluate expected policy, route, NAT, WAN, and session impact before commit.

Governing documents:

- `docs/CONFIGURATION-GOVERNANCE.md`
- `docs/POLICY-SIMULATION.md`
- `docs/MANAGEMENT-PLANE.md`

# High Availability

Initial FW HA is optional active/standby only.

Each node retains its own Appliance ID, physical NIC/MAC identity, management/history/HA identity, local storage, clock state, certificates, source journals, and capture provenance.

The pair has a stable Cluster ID and cluster-owned virtual forwarding identity.

> **Virtual network identity is forwarding identity, not capture identity.**

Loss of peer communication is not proof the peer is dead. Promotion requires a later explicit fencing/election contract. Net-Hunter is not an HA quorum/witness dependency.

Secure-access and tunnel continuity require separate qualification; ordinary forwarding failover does not prove WireGuard Access Sessions or L2TPv3/IPsec tunnels remained uninterrupted.

# Retention, Holds, Archive, and Destruction

PCAP, journal domains, and future inspection data have separately governed retention.

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

Holds override ordinary expiration. Storage pressure does not invent deletion authority.

# Appliance Lifecycle, Recovery, Encryption, and Platform Trust

Stronghold distinguishes:

```text
REBUILDABLE PLATFORM
RECOVERABLE CONFIGURATION
APPLIANCE / TRUST IDENTITY
AUTHORITATIVE HISTORY
DERIVED STATE
TRANSIENT RUNTIME STATE
```

A successful package install or boot is not sufficient update validation. Signed offline updates are required for restricted environments. HA update direction is standby-first and stops if the standby cannot be validated.

FW storage direction uses encrypted block devices beneath Btrfs/XFS. Hunter favors purpose-separated ZFS-native encrypted datasets. TPM/HSM mechanisms may protect normal key use but must not become the sole recovery path for authoritative history.

Secure Boot, measured boot, and attestation remain separate facts. Development, qualified, and supported hardware remain distinct states.

# Hardware and Performance Qualification

Stronghold claims apply to qualified profiles, not arbitrary systems that boot.

Qualification binds relevant combinations of:

```text
Stronghold release
CPU / architecture
RAM / ECC requirements
NIC model / driver / firmware
AF_XDP / XDP operating mode
queue / RSS / UMEM configuration
PCIe / NUMA topology
storage
platform trust state
enabled feature set
```

Performance claims include both throughput and PPS. The 64-byte/high-PPS case is a required stress dimension, not just large-frame bandwidth testing.

Capture-only throughput is not reused as a full-feature firewall, HA, inspection, or secure-access claim.

# IDS / TLS Inspection

Future TLS/application inspection is architecture-defined but implementation-deferred.

Authoritative original encrypted-wire PCAP remains the packet-history authority. Where inspection is explicitly selected, decrypted payload exists only in controlled volatile processing on FW and is transported over a purpose-specific mutually authenticated encrypted INSPECTION path to the future Hunter IDS/Inspection Jail.

Stronghold FW never intentionally persists decrypted inspection payload at rest.

`docs/IDS-INSPECTION.md` governs future inspection architecture.

# Observability and Support

Health is domain-specific. A healthy interface/process does not prove capture, durability, forwarding, journal, Hunter, Access, Agent, secure-access, or inspection health.

External syslog/SNMP/API systems consume Stronghold state but do not become authority for it.

Stronghold has no permanent vendor backdoor. Support collection, remote support, engineering access, and vendor BMC/OOB access remain separately controlled and attributable.

Governing documents:

- `docs/OBSERVABILITY.md`
- `docs/SUPPORT-DIAGNOSTICS.md`

# Resource Priority

On Stronghold FW:

```text
1. AF_XDP packet acquisition / ring service
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation / decision / journal append
5. HA heartbeat/control reserved lightweight capacity
6. journal checkpoint/finalization
7. local tier movement / backlog
8. HA bulk state synchronization
9. authoritative Net-Hunter HISTORY transfer
10. compression where approved
11. deeper analytics / observability / support / IDS / secure-access housekeeping
```

Backlog recovery, journal export, analytics, support work, IDS, and control-plane housekeeping may lag or throttle before live capture is sacrificed.

# Implementation Boundaries

Stronghold Access implementation belongs under:

```text
go/access/
```

Stronghold Agent implementation belongs under:

```text
go/agent/
```

Direct imports of another product's internal implementation packages are prohibited. A shared protocol/schema package may exist only after the cross-product contract is frozen, versioned, and explicitly owned.

# Current Scope

Implementation begins with **Phase 0 — Traffic Observation Foundation**.

Phase 0 uses AF_XDP and proves the physical observation foundation before later subsystems are implemented.

Secure Access, Stronghold Agent endpoint enforcement, IDS/IPS, full firewall enforcement, HA, recovery/DR, complete cryptographic journals, and other later systems remain outside Phase 0 even when their architecture is already defined.

`docs/ROADMAP.md` governs implementation sequencing.

# Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

A feature is incomplete if it cannot distinguish observation, authorization, attempted action, actual result, failure, gap, degraded coverage, and remaining recoverable history.

# Governing Architecture Index

```text
docs/PROJECT-BOUNDARIES.md        product/implementation ownership
docs/AF-XDP.md                    FW packet acquisition
docs/STATEFUL-ENFORCEMENT.md      FW stateful enforcement
docs/CONFIGURATION-GOVERNANCE.md  configuration authority
docs/POLICY-SIMULATION.md         pre-commit simulation
docs/MANAGEMENT-PLANE.md          management authority/surfaces
docs/JOURNAL-EXPORT.md            SIEM/log export
docs/NET-HUNTER-RECORDS.md        Hunter records/search/reprocessing
docs/OBSERVABILITY.md             health/alerting
docs/SUPPORT-DIAGNOSTICS.md       support/vendor OOB
docs/IDS-INSPECTION.md            future TLS/IDS inspection
docs/SECURE-ACCESS.md             earlier detailed secure-access model
docs/access/ARCHITECTURE.md       authoritative Stronghold Access product
docs/agent/ARCHITECTURE.md        authoritative Stronghold Agent product
docs/ROADMAP.md                   implementation sequencing
```
