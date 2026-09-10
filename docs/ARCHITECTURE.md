# Stronghold Architecture

## Purpose

This document records the integrated high-level architecture of the **Stronghold platform**. Focused subsystem documents govern their respective implementation details.

> **Stronghold is an enforcement system that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.**

Stronghold is **one tightly coupled security platform**, not a collection of unrelated products.

The platform consists of three infrastructure components plus the endpoint Agent component:

```text
Stronghold FW
    physical observation / enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    third Stronghold infrastructure node
    supported customer VM or bare-metal server
    access-control / identity / authorization coordination

Stronghold Agent
    endpoint component / endpoint Policy Enforcement Point
    managed through Stronghold Access
```

The component model is governed by `docs/PROJECT-BOUNDARIES.md`.

> **Tightly coupled platform does not mean monolithic software.**

Components communicate through explicit authenticated/versioned Stronghold contracts while retaining distinct implementation ownership, authority, failure domains, and qualification boundaries.

```text
OBSERVE → RECORD → ENFORCE → REMEMBER → HUNT
```

> **Observe the truth. Preserve the history. Explain the decision.**

# Truth Model

Stronghold separates three categories of truth:

```text
WHAT WAS PRESENTED
    authoritative physical-interface observation / PCAPNG

WHAT STRONGHOLD DID
    authoritative operational and decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation, correlation, enrichment, and intelligence context
```

## Observation Survives Interpretation

If Stronghold preserves a frame it cannot decode or classify today, that packet remains authoritative history. Later decoders, Net-Hunter reprocessing, IDS findings, or Pathfinder intelligence may create new derived interpretation without rewriting the original observation.

```text
observed then
!=
interpreted then

historically derivable now
!=
known then
```

Missing interpretation must never erase observation, and missing history must never be presented as proof of absence.

# Governing Principles

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

# Common Truth Boundaries

```text
route available                     != route authorized
certificate valid                   != peer authorized
packet not decoded                  != packet not observed
no journal record                   != event did not occur
timestamp precision                 != timestamp accuracy
policy allowed                      != forwarding succeeded
object expired                      != destruction authorized
bytes received                      != history durably committed
archived                            != destroyed
new interpretation                  != new historical observation
forwarding HA succeeded             != capture continuity guaranteed
virtual cluster identity            != physical capture identity
software installed                  != Stronghold update validated
configuration restored              != appliance validated
backup written                      != backup proven restorable
ZFS redundancy                      != disaster recovery
disk encrypted                      != secrets safely designed
TPM protected                       != disaster recoverable
entry written                       != journal entry durably committed
signed journal checkpoint           != physically immutable storage
system booted                       != approved system state
hardware detected                   != hardware supported
link speed                          != validated dataplane rate
capture throughput                  != full-feature firewall throughput
storage capacity                    != sustainable ingest capacity
AF_XDP available                    != AF_XDP zero-copy available
AF_XDP zero-copy available          != Stronghold zero-copy qualified
WireGuard peer authenticated        != user authenticated
tunnel established                 != resource authorized
802.1X authenticated                != resource authorized
RADIUS Access-Accept                != Stronghold Access GRANT
endpoint ALLOW                      != FW ALLOW
endpoint DENY                       != FW observed DENY
device on VLAN                      != device authorized for VLAN/zone
policy received                     != policy activated
policy activated                    != endpoint enforcement healthy
L2TPv3 tunnel established           != site authorized
IPsec SA established                != traffic authorized
Pathfinder record exists            != observable malicious
Pathfinder malicious classification != compromise proven
Pathfinder confidence HIGH          != Stronghold action authorized
Pathfinder match                    != IDS detection
new Pathfinder interpretation       != original historical knowledge
```

# Integrated Platform Topology

```text
                              PATHFINDER
                       ISS threat intelligence
                              authority
                                 |
                     authenticated/versioned
                       intelligence exchange
                                 |
         +-----------------------+-----------------------+
         |                       |                       |
         v                       v                       v
+------------------+    +-------------------+    +----------------------+
| STRONGHOLD ACCESS|    |   STRONGHOLD FW   |    | STRONGHOLD NET-HUNTER|
| Server / VM      |<-->| Physical Appliance|--->| Physical Appliance   |
| PE / PA          |    | AF_XDP + PEP      |    | History / Hunt       |
| 802.1X / AAA     |    | Capture / FW      |    | Reprocess / IDS      |
+---------+--------+    +---------+---------+    +----------------------+
          |                       ^
          | signed policy         |
          v                       | normal or WireGuard traffic
 +------------------+             |
 | STRONGHOLD AGENT |-------------+
 | Windows endpoint |
 | Go + WFP PEP     |
 +------------------+
```

Pathfinder is not a Stronghold component. It is a separate ISS threat-intelligence system that may enrich the Stronghold environment through a controlled first-class integration.

# Stronghold FW

## Platform Direction

```text
Platform:           Arch Linux
Architecture:       x86_64
Administration:     CLI first
NICs:               qualified physical PCIe Ethernet
Packet acquisition: AF_XDP
Capture format:     PCAPNG
Enforcement:        nftables/netfilter direction
OS storage:         dedicated SSD / Btrfs direction
HOT capture:        NVMe / XFS direction
WARM/backlog:       SSD / XFS direction
1 GbE:              current qualification target
10 GbE:             primary production target
40 GbE:             future target only
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

## Local Working History

Stronghold FW remains operational and useful without Net-Hunter or while Hunter is unavailable.

```text
OPERATOR WORKING SET
    approximately 30–60 minutes
    near-real-time traffic / policy / route / NAT / WAN visibility

HUNTER OUTAGE BUFFER
    approximately 4 hours without Hunter is CRITICAL
    qualified physical capacity direction: roughly 8 hours equivalent ingest
```

The administrator should be able to determine what the FW is doing now without waiting for Hunter indexing.

Backlog recovery is subordinate to live observation. Catch-up must not starve current packet acquisition or active PCAP writes.

## Pathfinder Context on FW

Stronghold FW may display or consume approved Pathfinder-derived intelligence alongside recent traffic and policy facts.

The distinction is explicit:

```text
STRONGHOLD
    what was observed
    what policy matched
    what route/WAN/NAT was selected
    what action actually occurred

PATHFINDER
    what an observable is currently interpreted to represent
    confidence / source / campaign / relationship context
```

Pathfinder context may later be an explicit Stronghold policy input, but Pathfinder itself does not directly command the dataplane.

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

Source journals remain Stronghold authority. External systems receive copies/projections and do not replace the authoritative source.

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

Net-Hunter is the historical Stronghold appliance.

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

Net-Hunter provides long-duration packet history, deep historical search, reprocessing, configuration backup, and forensic reconstruction.

Defined application jails:

1. **PCAP Data Ingest Jail** — authenticated receive, verification, durable commit, ACK.
2. **Record Processing Jail** — derives/rebuilds searchable records from authoritative history.
3. **External User Interface Jail** — read-only investigation of authoritative history with non-authoritative workspace.
4. **FW Configuration Backup Jail** — isolated versioned FW configuration history.
5. **IDS / Inspection Jail** — future/optional isolated deep-inspection processing.

The FreeBSD host owns ZFS, raw storage/HBA authority, host networking, jail lifecycle, updates, and encryption-key authority.

Derived records/indexes are rebuildable. They must never become the sole proof that authoritative traffic existed.

## Pathfinder Retrospective Enrichment

New Pathfinder intelligence can be applied to old Net-Hunter history.

```text
new Pathfinder interpretation
        ↓
observable / relationship updated
        ↓
Net-Hunter historical matching / reprocessing
        ↓
prior endpoints / sessions / flows / PCAP references
```

This is one of the major reasons to preserve authoritative source history independently of current interpretation.

```text
retrospective match
!=
historical real-time detection
```

`docs/NET-HUNTER-RECORDS.md` governs Hunter records/search/reprocessing and `docs/PATHFINDER-INTEGRATION.md` governs Pathfinder lineage.

# Stronghold Access

Stronghold Access is the **third Stronghold infrastructure component** when deployed.

It runs on a supported customer VM or supported bare-metal server on the customer's network rather than inside the high-PPS FW dataplane.

It owns:

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
optional Pathfinder intelligence / risk inputs
```

Stronghold Access does not own FW packet capture, routing, NAT, physical observation, or FW Traffic Decision Journal authority.

`docs/access/ARCHITECTURE.md` is the authoritative Access architecture.

## 802.1X as a First-Class Control

802.1X is a first-class Stronghold security primitive, not an afterthought.

```text
network admission
!=
resource authorization
```

EAP-TLS is an important high-assurance direction where supported. MAB may be necessary for legacy/non-supplicant devices but is not equivalent assurance to authenticated 802.1X.

External AAA may provide identity/authorization attributes, but those attributes remain inputs to Stronghold Access policy rather than hidden authorization bypasses.

## Network Context and Agent Policy Distribution

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

The Agent cannot self-promote into a trusted VLAN/zone merely by reporting local state.

## Pathfinder Risk / Trust Input

Pathfinder may provide an intelligence/risk input to Stronghold Access reevaluation.

Potential policy outcomes may later include MFA reauthentication, narrower resource scope, shorter authorization leases, Access Session revocation, or network-admission reevaluation.

```text
Pathfinder risk signal
!=
automatic Access Session revocation
```

Stronghold Access remains the authority over the actual authorization result.

# Stronghold Agent

Stronghold Agent is the endpoint component of Stronghold and the endpoint Policy Enforcement Point. Windows is the initial platform direction.

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
    ├── DENY → endpoint decision record
    └── ALLOW
            │
            ▼
       network/tunnel
            │
            ▼
       Stronghold FW
            │
            ▼
       independent FW policy
```

Endpoint ALLOW cannot force FW ALLOW. Endpoint DENY is not represented as a FW denial if traffic never reached the FW.

`docs/agent/ARCHITECTURE.md` is the authoritative Agent architecture.

## Agent / Pathfinder Boundary

Stronghold Agent normally does not query Pathfinder directly.

```text
Pathfinder
    ↓
Stronghold Access policy evaluation
    ↓
signed Stronghold Agent policy
    ↓
Stronghold Agent
```

The endpoint receives the authorized Stronghold policy result rather than becoming an independent threat-intelligence consumer.

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

`docs/SECURE-ACCESS.md` retains detailed secure-access concepts; where component ownership differs, `docs/PROJECT-BOUNDARIES.md`, `docs/access/ARCHITECTURE.md`, and `docs/agent/ARCHITECTURE.md` take precedence.

# IDS / TLS Inspection

IDS/TLS inspection is architecture-defined but implementation-deferred.

Stronghold FW retains authoritative encrypted-wire PCAP. Where inspection is explicitly selected, plaintext exists only in controlled volatile processing on the FW and is transported over a separate authenticated encrypted INSPECTION path to the isolated Hunter IDS/Inspection Jail.

Stronghold FW never intentionally persists decrypted inspection payload at rest.

## Pathfinder Complement

Pathfinder complements IDS/IPS rather than replacing the detector or ruleset.

```text
Stronghold observation
        ↓
IDS detection
        ├───────────────┐
        │               │
        ▼               ▼
content/protocol     Pathfinder intelligence
        │               │
        └───────┬───────┘
                ▼
        Stronghold correlation
```

A suspicious protocol/content result may become more actionable when combined with Pathfinder classification, confidence, infrastructure/campaign relationships, certificate/hash context, or historical intelligence.

```text
IDS finding + Pathfinder match
!=
automatic IPS authority
```

Any IPS action remains a Stronghold policy decision and must record the actual enforcement result.

`docs/IDS-INSPECTION.md` governs the inspection boundary and `docs/PATHFINDER-INTEGRATION.md` governs intelligence authority.

# Pathfinder Integration

Pathfinder is a separate ISS system and threat-intelligence authority.

> **Pathfinder owns the organization's record and interpretation of threat intelligence. Stronghold owns observation, access/enforcement decisions, operational history, and the actions performed in the Stronghold environment.**

Stronghold may consume Pathfinder intelligence for:

```text
FW recent traffic / operator enrichment
explicit future policy inputs
IDS/IPS context
Net-Hunter retrospective matching / reprocessing
Stronghold Access risk / trust evaluation
```

Stronghold may also submit qualified observables and relationships to Pathfinder for independent interpretation.

Neither system rewrites the other's source truth.

Pathfinder is not required for basic AF_XDP capture, ordinary FW forwarding, local journal commit, Hunter history ingest, valid Agent WFP enforcement, or ordinary Access operation unless a customer deliberately configures a policy requiring fresh Pathfinder context.

`docs/PATHFINDER-INTEGRATION.md` governs this relationship.

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

Policy simulation/counterfactual testing is intended to let administrators evaluate expected policy, route, NAT, WAN, session, secure-access, and later intelligence-driven impact before commit.

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

Secure Boot, measured boot, and attestation remain separate facts. Development, qualification, and supported hardware remain distinct states.

# Hardware and Performance Qualification

Stronghold claims apply to qualified profiles, not arbitrary systems that boot.

Qualification binds relevant combinations of:

```text
Stronghold release
CPU / architecture
RAM / ECC requirement
NIC / driver / firmware
AF_XDP mode
queue / RSS / UMEM configuration
PCIe / NUMA topology
storage
feature profile
```

Performance testing includes Gbps and PPS, representative packet sizes, bidirectional traffic, flow counts, sustained duration, storage contention, and truthful loss accounting.

The 64-byte packet case is a first-class stress profile; a clean large-frame throughput result is not enough.

# Resource Priority

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
11. IDS / Pathfinder / analytics / observability / support / Access housekeeping
```

Secondary work may degrade or lag. It must not silently win over authoritative live observation.

Pathfinder lookup is not an excuse to introduce an unbounded synchronous per-packet dependency on the FW dataplane.

# Current Implementation Scope

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

Stronghold Access implementation, Agent endpoint enforcement, Pathfinder integration, IDS/IPS, secure access, HA, full firewall enforcement, and other later systems remain outside Phase 0 even where their architecture is already defined.

# Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

A complete feature should preserve enough state/history to answer:

```text
what was observed?
what was known?
what was unknown?
what was decided?
why?
what was actually performed?
what was NOT_PERFORMED?
what failed?
what history remains?
what was lost or could not be established?
```

Pathfinder-enriched features add one more requirement: Stronghold must also be able to establish **which Pathfinder record/interpretation influenced the result and whether that interpretation existed at the time or was applied retrospectively later.**

# Governing Documents

Focused architecture includes:

```text
docs/PROJECT-BOUNDARIES.md
docs/AF-XDP.md
docs/JOURNAL-EXPORT.md
docs/NET-HUNTER-RECORDS.md
docs/CONFIGURATION-GOVERNANCE.md
docs/STATEFUL-ENFORCEMENT.md
docs/POLICY-SIMULATION.md
docs/MANAGEMENT-PLANE.md
docs/OBSERVABILITY.md
docs/SUPPORT-DIAGNOSTICS.md
docs/SECURE-ACCESS.md
docs/access/ARCHITECTURE.md
docs/ENDPOINT-ENFORCEMENT.md
docs/agent/ARCHITECTURE.md
docs/IDS-INSPECTION.md
docs/PATHFINDER-INTEGRATION.md
docs/ROADMAP.md
```

Where older secure-access/endpoint documents conflict with current component ownership, `docs/PROJECT-BOUNDARIES.md`, `docs/access/ARCHITECTURE.md`, and `docs/agent/ARCHITECTURE.md` govern.
