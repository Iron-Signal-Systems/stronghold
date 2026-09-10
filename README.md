# Stronghold

**Stronghold by Iron Signal Systems**

Stronghold is an enforcement platform that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.

> **Observe the truth. Preserve the history. Explain the decision.**

Stronghold is **one tightly coupled security platform**, not a collection of unrelated products.

The platform currently consists of three infrastructure components plus the endpoint Agent component:

```text
Stronghold FW
    physical observation / enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    third Stronghold infrastructure node
    supported server or VM on the customer's network
    access-control / identity / authorization coordination
    synchronized endpoint-policy distribution / convergence

Stronghold Agent
    endpoint component / endpoint Policy Enforcement Point
    managed through Stronghold Access
    enforces the endpoint-applicable projection of finalized Stronghold policy
```

The components cooperate through explicit authenticated/versioned Stronghold contracts. They are intentionally separate implementations and failure domains, but they operate as one Stronghold system.

> **Tightly coupled platform does not mean monolithic software.**

Stronghold uses one governed policy authority with multiple enforcement projections. Stronghold Access coordinates the device-specific policy projection delivered to managed Agents so that known-denied endpoint traffic can be rejected before it becomes network traffic, while Stronghold FW independently enforces traffic that is actually presented to it.

A managed endpoint continues to enforce its active endpoint policy when it leaves the organization's network, subject to explicit policy-generation, lease, holdover, expiration, and revocation semantics.

See [`docs/PROJECT-BOUNDARIES.md`](docs/PROJECT-BOUNDARIES.md) and [`docs/DISTRIBUTED-POLICY-ENFORCEMENT.md`](docs/DISTRIBUTED-POLICY-ENFORCEMENT.md).

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

Observation survives interpretation. A packet Stronghold cannot decode today remains authoritative history if it was actually captured. Better future decoders or intelligence may create new derived interpretation without rewriting the original observation.

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Where a managed Agent can definitively deny from its active Stronghold policy, reject before transmission rather than intentionally send garbage to the FW merely to deny it again.**

> **An endpoint-local DENY must be reported as an endpoint decision, never fabricated as a Stronghold FW observation or FW denial.**

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
route available                    != route authorized
certificate valid                  != peer authorized
packet not decoded                 != packet not observed
no journal record                  != event did not occur
timestamp precision                != timestamp accuracy
policy allowed                     != forwarding succeeded
bytes received                     != history durably committed
archived                           != destroyed
forwarding HA succeeded            != capture continuity guaranteed
virtual cluster identity           != physical capture identity
software installed                 != Stronghold update validated
configuration restored             != appliance validated
backup written                     != backup proven restorable
ZFS redundancy                     != disaster recovery
disk encrypted                     != secrets safely designed
TPM protected                      != disaster recoverable
link speed                         != validated dataplane rate
capture throughput                 != full-feature firewall throughput
node performance                   != HA cluster performance
storage capacity                   != sustainable ingest capacity
AF_XDP available                   != AF_XDP zero-copy available
zero-copy available                != Stronghold zero-copy qualified
WireGuard peer authenticated       != user authenticated
tunnel established                 != resource authorized
802.1X authenticated               != resource authorized
RADIUS Access-Accept               != Stronghold Access GRANT
endpoint ALLOW                     != FW ALLOW
endpoint DENY                      != FW observed DENY
endpoint DENY                      != packet transmitted
endpoint attempt reported          != FW observed packet
device on VLAN                     != device authorized for VLAN/zone
Agent policy projection            != complete FW configuration
policy generation finalized        != every Agent synchronized
policy received                    != policy activated
policy activated                   != endpoint enforcement healthy
Pathfinder match                   != Stronghold enforcement action
Pathfinder malicious classification != compromise proven
new Pathfinder interpretation      != knowledge Stronghold had at observation time
```

# Stronghold FW

Stronghold FW is the live-network appliance and network-side Policy Enforcement Point.

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

## Capture Requirement — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline.

Stronghold FW uses **AF_XDP as the intended packet-acquisition foundation**. Queue/RSS behavior, UMEM/rings, copy/native/zero-copy state, NIC/driver/firmware behavior, drop accounting, CPU affinity, NUMA placement, PCIe topology, packet size, and PPS are qualified rather than assumed.

See [`docs/AF-XDP.md`](docs/AF-XDP.md).

## Enforcement Model

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

## Standalone FW and Local Working History

Stronghold FW remains a useful firewall when Net-Hunter is not deployed or temporarily unavailable.

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

# Journals and External Log Integration

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

# Stronghold Net-Hunter

Net-Hunter is the historical Stronghold appliance. It runs on FreeBSD with ZFS and jails and owns long-term historical responsibilities:

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
future Pathfinder retrospective enrichment
```

Net-Hunter is optional for basic FW operation, but it is what turns Stronghold from a short-window enforcement appliance into a durable historical and forensic platform.

Defined application jails include:

1. **PCAP Data Ingest Jail**
2. **Record Processing Jail**
3. **External User Interface Jail**
4. **FW Configuration Backup Jail**
5. **IDS / Inspection Jail** — future/optional

The External UI remains read-only with respect to authoritative history. Derived indexes, intelligence correlations, and interpretations may be rebuilt without rewriting authoritative PCAP or source journals.

See [`docs/NET-HUNTER-RECORDS.md`](docs/NET-HUNTER-RECORDS.md).

# Stronghold Access

Stronghold Access is the **third Stronghold infrastructure component** when deployed. It runs as a supported VM or supported bare-metal server on the customer's network rather than inside the Stronghold FW high-rate dataplane.

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
endpoint policy-generation synchronization
Agent check-in / fleet convergence state
controlled Stronghold FW integration
future Pathfinder risk/intelligence inputs
```

The intended relationship is tightly coupled but authority-preserving:

```text
FINALIZED STRONGHOLD POLICY GENERATION
        │
        ├──────────────► Stronghold FW network projection / PEP
        │
        ▼
Stronghold Access
        │
        │ device-specific signed endpoint projection
        ▼
Stronghold Agent endpoint PEP
```

A newly finalized generation should be made available promptly to connected Agents. If an Agent misses the update because it is offline, asleep, or disconnected, the next authenticated check-in compares the Agent's active generation with the currently required generation and delivers the applicable update when needed.

```text
policy finalized
!=
every Agent synchronized

policy sent
!=
policy activated
```

Remote zero-trust access follows the NIST SP 800-207 PE/PA/PEP direction with WireGuard as the secure transport/data plane. WireGuard does not become the authorization system.

Site-to-site / branch-office tunneling is a separate architecture using L2TPv3 protected by IPsec and normal Stronghold policy around the tunnel.

See [`docs/access/ARCHITECTURE.md`](docs/access/ARCHITECTURE.md), [`docs/DISTRIBUTED-POLICY-ENFORCEMENT.md`](docs/DISTRIBUTED-POLICY-ENFORCEMENT.md), and [`docs/SECURE-ACCESS.md`](docs/SECURE-ACCESS.md).

# Stronghold Agent

Stronghold Agent is the endpoint component of the Stronghold platform and the endpoint Policy Enforcement Point. Windows is the initial platform direction, with implementation reserved under `go/agent/`.

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

## Synchronized Endpoint Policy and Off-Network Enforcement

Stronghold Agent does not maintain an unrelated endpoint-firewall policy universe. It enforces the **endpoint-applicable projection of the finalized Stronghold policy generation** distributed and coordinated through Stronghold Access.

For example:

```text
FINALIZED GENERATION 844

Policy 798
    Device:       FIN-PC-17
    User:         DOMAIN\John
    Application:  powershell.exe
    Destination:  PAYROLL-DB
    Service:      TCP/1433
    Action:       DENY
```

On the endpoint:

```text
powershell.exe
      │
      │ attempts PAYROLL-DB:1433
      ▼
Stronghold Agent
      │
      │ Generation 844 / Policy 798
      ▼
    DENY
      │
      ├── endpoint decision reported
      └── packet not transmitted
```

Stronghold FW does not need to spend dataplane resources denying traffic the managed Agent has already definitively rejected. The platform may show the endpoint denial in a unified operator view, but the event remains truthful:

```text
DENIED AT ENDPOINT
Policy 798
Packet presented to FW: NO
```

The active endpoint policy remains applicable when the managed device leaves the organization's network. Office LAN, home Wi-Fi, hotel Wi-Fi, or a mobile hotspot do not independently erase a Stronghold endpoint restriction.

If a device misses a new generation, Stronghold Access detects the mismatch at a later authenticated check-in and supplies the required signed projection according to the defined synchronization/lease contract.

The FW remains independently authoritative for traffic that does leave the endpoint:

```text
Agent DENY
    -> do not transmit

Agent ALLOW
    -> connection may be attempted
    -> FW still independently evaluates traffic actually presented to it
```

See [`docs/DISTRIBUTED-POLICY-ENFORCEMENT.md`](docs/DISTRIBUTED-POLICY-ENFORCEMENT.md).

Network-side VLAN/zone context is stronger than endpoint self-assertion. An Agent cannot declare itself trusted because it reports a particular VLAN or network.

The initial Agent direction is application/process-aware connection enforcement, not general endpoint deep-payload inspection. Any future local proxy or kernel-mode WFP callout-driver design is separately qualified.

Protected Endpoint mode may later force eligible traffic from selected high-value systems through an authenticated encrypted Stronghold transport while still preserving necessary local-link traffic and truthful metadata visibility.

See [`docs/agent/ARCHITECTURE.md`](docs/agent/ARCHITECTURE.md).

# 802.1X and Multiple Enforcement Points

802.1X is a first-class Stronghold Access security primitive, not an afterthought.

A mature Stronghold deployment may include:

```text
NETWORK-ADMISSION PEP
    switch / AP / 802.1X

ENDPOINT PEP
    Stronghold Agent / WFP

NETWORK PEP
    Stronghold FW
```

A decision at one enforcement location does not fabricate a decision at another.

# Secure Access

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

# Pathfinder Intelligence Integration

Iron Signal Systems **Pathfinder** is a separate ISS threat-intelligence system that may act as a first-class intelligence/enrichment peer to Stronghold.

Pathfinder is not a fifth Stronghold component and is not a replacement for Stronghold policy, IDS/IPS, or authoritative packet history.

> **Pathfinder owns the organization's record and interpretation of threat intelligence. Stronghold owns observation, access/enforcement decisions, operational history, and the actions performed in the Stronghold environment.**

Conceptually:

```text
                         PATHFINDER
                  threat intelligence authority
                            │
                authenticated/versioned exchange
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
   STRONGHOLD FW      STRONGHOLD ACCESS   NET-HUNTER
   traffic context    risk/trust input     retrospective matching
   policy context     reevaluation         historical correlation
   IDS/IPS context                         reprocessing
```

Stronghold may also submit qualified observables to Pathfinder. Submission does not itself classify an observable as malicious; Pathfinder retains interpretation authority.

Important boundaries:

```text
Pathfinder match                    != Stronghold enforcement action
Pathfinder malicious classification != compromise proven
Pathfinder risk signal              != automatic Access revocation
IDS finding                         != Pathfinder intelligence confirmed
Stronghold observation              != Pathfinder intelligence
new Pathfinder interpretation      != original historical knowledge
```

This creates an important historical capability: new Pathfinder intelligence can be applied to old Net-Hunter history without rewriting what Stronghold originally observed or knew.

Stronghold Agent normally receives the resulting authorized policy from Stronghold Access rather than communicating directly with Pathfinder.

See [`docs/PATHFINDER-INTEGRATION.md`](docs/PATHFINDER-INTEGRATION.md).

# IDS / TLS Inspection

Future selective TLS/application inspection is architecture-defined but implementation-deferred.

The FW retains authoritative encrypted-wire PCAP. Where inspection is explicitly selected, plaintext exists only in controlled volatile processing on the FW and is transported over a separate authenticated encrypted INSPECTION path to a future isolated Hunter IDS/Inspection Jail.

Stronghold FW never intentionally persists decrypted inspection payload at rest.

Pathfinder is intended to complement IDS/IPS with intelligence context such as observable classification, confidence, campaign/infrastructure association, and historical relationships. IDS findings and Pathfinder intelligence remain separate sources and neither independently creates enforcement authority.

See [`docs/IDS-INSPECTION.md`](docs/IDS-INSPECTION.md).

# Configuration, Simulation, and Management

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

A finalized generation may produce different enforcement projections for FW and managed Agents while remaining one governed policy authority. Finalization does not prove every Agent has synchronized; convergence state must remain explicit.

Policy simulation/counterfactual testing is intended to show expected policy, route, NAT, WAN, session, secure-access, and later intelligence-driven impact before activation.

See [`docs/CONFIGURATION-GOVERNANCE.md`](docs/CONFIGURATION-GOVERNANCE.md), [`docs/POLICY-SIMULATION.md`](docs/POLICY-SIMULATION.md), and [`docs/MANAGEMENT-PLANE.md`](docs/MANAGEMENT-PLANE.md).

# High Availability

Initial FW HA is optional **active/standby** only.

Each node retains its own Appliance ID, physical capture provenance, local storage, certificates, clock state, and node-local configuration. The cluster owns the virtual forwarding identity.

> **Virtual network identity is forwarding identity, not capture identity.**

Loss of peer communication is not, by itself, proof the peer is dead. Promotion requires a later explicit fencing/election contract. Net-Hunter is not an HA witness/quorum dependency.

# Recovery, Encryption, and Platform Trust

Stronghold separates rebuildable platform state, recoverable configuration, appliance/trust identity, authoritative history, derived state, and transient runtime state.

FW storage direction uses encrypted block devices beneath Btrfs/XFS. Hunter favors purpose-separated ZFS-native encrypted datasets. TPM/HSM mechanisms may protect normal key use but must not become the sole recovery path for authoritative history.

Secure Boot, measured boot, and attestation remain separate facts. Development hardware, qualification hardware, and supported production hardware remain distinct categories.

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
11. deeper analytics / observability / support / IDS / Access / Pathfinder housekeeping
```

Secondary work may degrade or lag. It must not silently win over authoritative live observation.

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

Stronghold Access implementation, Agent endpoint enforcement, distributed policy synchronization, Pathfinder integration, IDS/IPS, HA, full firewall enforcement, and other later systems remain outside Phase 0 even where their architecture is already defined.

See [`docs/ROADMAP.md`](docs/ROADMAP.md).

# Engineering Standard

Stronghold follows the Iron Signal Systems Engineering Standards pinned by `ENGINEERING-STANDARD`. Contributor behavior is governed by `AGENTS.md` and nested `AGENTS.md` files.

# Project Status

Stronghold is pre-release and under active development. Architecture and interfaces may change before a supported release.

# License

Stronghold is proprietary source-available software. See [`LICENSE`](LICENSE).

# Security

Do not report suspected vulnerabilities through public GitHub issues. See [`SECURITY.md`](SECURITY.md).