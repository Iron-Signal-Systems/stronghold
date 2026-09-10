# Stronghold Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation work within the Stronghold repository.

Stronghold is **one tightly coupled security platform** composed of multiple infrastructure and endpoint components. Component boundaries exist to preserve authority, implementation ownership, failure domains, qualification, and maintainability. They must not be interpreted as separate unrelated products.

> **Tightly coupled platform does not mean monolithic software.**

This file is a behavioral contract and does not replace focused governing architecture.

Read applicable documents before changing behavior:

```text
docs/PROJECT-BOUNDARIES.md
docs/ARCHITECTURE.md
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

Where older secure-access/endpoint summaries conflict with current component ownership, this precedence applies:

```text
docs/PROJECT-BOUNDARIES.md
docs/access/ARCHITECTURE.md
docs/agent/ARCHITECTURE.md
```

## Stronghold Platform Model

Stronghold consists of three infrastructure components plus the endpoint Agent component:

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
    endpoint component / endpoint PEP
    managed through Stronghold Access
```

Iron Signal Systems Pathfinder is **not** a Stronghold component. It is a separate ISS threat-intelligence system that may act as a first-class enrichment peer.

Component implementations may be separate processes/trees and must communicate through frozen, authenticated, versioned contracts rather than direct imports of another component's internal implementation packages.

## Governing Principles

> **Capture first. Never sacrifice observation for secondary work.**

> **AF_XDP is the intended Stronghold FW packet-acquisition foundation.**

> **Denied traffic should be cheap to reject, but never invisible.**

> **Path availability is not path permission.**

> **A secure tunnel is a protected path, not authorization to use every resource behind it.**

> **Endpoint ALLOW never grants network permission that Stronghold FW would otherwise deny.**

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

> **Expiration eligibility is not permission to delete.**

> **A full disk is a failure condition, not permission to rewrite history.**

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

> **Record what the system can actually establish, preserve the source of that truth, and never manufacture certainty beyond it.**

> **An incomplete index is not complete history.**

> **Stronghold has one management authority and multiple management surfaces.**

> **Health is domain-specific; one healthy subsystem must not hide failure in another.**

> **Stronghold has no permanent vendor backdoor.**

> **Decrypted inspection payload is never intentionally persisted at rest on Stronghold FW.**

> **Pathfinder intelligence may enrich Stronghold, but Pathfinder is not Stronghold enforcement authority.**

## Three Categories of Stronghold Truth

Preserve:

```text
WHAT WAS PRESENTED
    authoritative physical-interface PCAPNG

WHAT STRONGHOLD DID
    authoritative operational / decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation / correlation / enrichment
```

Pathfinder adds a separate external intelligence authority:

```text
WHAT PATHFINDER INTERPRETED
    threat-intelligence record / classification / relationship context
```

Never rewrite Stronghold source truth to make later interpretation look contemporaneous.

```text
observed then
!=
interpreted then

historically derivable now
!=
known then
```

## Common Truth Separations

Never collapse these distinctions:

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
key rotated                         != past exposure undone
entry written                       != journal entry durably committed
hash chain valid                    != checkpoint externally anchored
signed checkpoint                   != physically immutable storage
new journal epoch                   != uninterrupted history
system booted                       != approved system state
Secure Boot enabled                 != measured boot verified
measured boot available             != attestation performed
Appliance ID                        != TPM identity
hardware detected                   != hardware qualified
hardware compatible                 != hardware supported
link speed                          != validated dataplane rate
capture throughput                  != full-feature firewall throughput
node performance                    != HA cluster performance
storage capacity                    != sustainable ingest capacity
AF_XDP available                    != AF_XDP zero-copy available
AF_XDP zero-copy available          != Stronghold zero-copy qualified
queue configured                    != queue serviced adequately
worker running                      != AF_XDP ring healthy
database record                     != authoritative packet
no indexed result                   != no traffic
index unavailable                   != history unavailable
source history available            != index complete
reprocessing result                 != original knowledge
timeline                            != authoritative journal
exported PCAP                       != original PCAP segment
direct observation                 != correlated association
native OS change                    != Stronghold configuration commit
configured state                    != operational state
operational state                   != historical state
API access                          != commit permission
diagnostic action                   != repair action
process running                     != subsystem healthy
alert acknowledged                  != fault resolved
alert suppressed                    != event not recorded
syslog delivered                    != journal committed
support access                      != configuration authority
support access                      != history-destruction authority
Stronghold administration           != BMC administration
vendor-supported hardware           != Stronghold-qualified hardware
WireGuard peer authenticated        != user authenticated
user authenticated                  != device authorized
device authorized                   != resource authorized
WireGuard tunnel established        != resource authorized
AllowedIPs configured               != Stronghold resource authorization
Access Session created              != application connection succeeded
REVOKE issued                       != revocation fully enforced
802.1X authenticated                != resource authorized
RADIUS Access-Accept                != Stronghold Access GRANT
MAB admitted                        != 802.1X authenticated
endpoint ALLOW                      != FW ALLOW
endpoint DENY                       != FW observed DENY
Agent reports VLAN/zone             != network-side VLAN/zone established
application identified              != application trusted
originating process identified      != payload decoded
L2TPv3 tunnel established           != site authorized
IPsec SA established                != traffic authorized
remote network reachable            != remote network permitted
Pathfinder record exists            != observable malicious
Pathfinder malicious classification != compromise proven
Pathfinder confidence HIGH          != Stronghold action authorized
Pathfinder match                    != IDS detection
IDS finding                         != Pathfinder confirmation
Pathfinder risk signal              != Access Session revoked
Pathfinder unavailable              != observable trusted
new Pathfinder interpretation       != original historical knowledge
retrospective Pathfinder match      != historical real-time detection
```

# Stronghold FW Rules

## Capture

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Wireshark/dumpcap on the same supported physical interface under equivalent supported conditions is the reference visibility baseline.

Unknown EtherTypes, unknown IP protocols, malformed/vendor-specific traffic, and Layer-2/control-plane traffic are not discarded merely because Stronghold does not understand them.

RAM/UMEM is buffering only, not durable history.

`docs/AF-XDP.md` defines the governing acquisition architecture.

AF_PACKET/TPACKET_V3 is not the planned primary Phase 0 implementation and must not be introduced as a silent fallback that preserves an AF_XDP qualification claim.

AF_XDP availability, native-driver operation, zero-copy capability, and Stronghold qualification are separate facts.

Queue/RSS topology, UMEM/rings, worker affinity, CPU/NUMA/PCIe locality, NIC/driver/firmware behavior, offload representation, timestamps, and drop accounting must be measured and exposed truthfully.

Keep the initial XDP program minimal and measurable. XDP/eBPF does not become a second firewall policy engine before the authoritative capture path is proven.

## FW Networking / Authorization

Stronghold is explicit-allow/default-deny.

```text
observe / preserve
    ↓
establish required policy facts
    ↓
authorize
    ├── deny / no allow → drop
    └── authorized to proceed
             ↓
       bridge / route
             ↓
 WAN / NAT / state / deeper work
             ↓
          final egress
             ↓
      journal actual result
```

Route availability, NAT, FQDN association, WAN reachability, HA membership, IDS availability, WireGuard peer state, L2TPv3 state, IPsec SA state, or Pathfinder classification never independently grants security permission.

A route/FIB lookup may establish a needed fact but does not authorize.

Policy uses dense integer positions, lowest-to-highest, first match wins. Policy ID is stable identity only, never priority.

Preserve `NOT_PERFORMED` when later pipeline work did not occur.

## FW Recent Working Set and Hunter Outage

The FW local working/history model has two roles:

```text
NORMAL OPERATOR WORKING SET
    approximately 30–60 minutes
    near-real-time traffic / policy / route / NAT / WAN visibility

HUNTER OUTAGE BUFFER
    approximately 4 hours without Hunter = CRITICAL
    qualified capacity direction ≈ 8 hours equivalent aggregate ingest
```

The 4-hour threshold is not deletion permission.

Backlog recovery never outranks live packet acquisition or active PCAP writes.

## FW Resource Priority

Conceptual priority:

```text
1. AF_XDP packet acquisition / ring service
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation / decision / journal append
5. HA heartbeat/control reserved lightweight capacity
6. journal checkpoint/finalization
7. local tier movement/backlog
8. HA bulk state sync
9. authoritative Hunter HISTORY transfer
10. compression where approved
11. IDS / Pathfinder / Access housekeeping / analytics / observability / support
```

Secondary work must throttle before live observation is sacrificed or hidden.

Pathfinder lookup must not become an unbounded synchronous per-packet dependency.

# Journals and Export

Maintain independent append-oriented journals:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

Each journal has separate identity/domain, origin, epoch, monotonic sequence, and tamper-evident linkage according to its contract.

`write()` success is not durable journal commit.

Hash/signature calculation requires a canonical exact-byte representation.

Normal reboot continues a verified journal. Catastrophic continuity failure creates an explicit gap/new epoch rather than pretending continuity.

Hunter preserves FW source chains without rewriting them.

External SIEM/log export is a projection/copy, not journal authority.

```text
journal committed locally != SIEM event delivered
SIEM event delivered      != Hunter verified durable commit
not exported              != not journaled
SIEM export gap           != Stronghold journal gap
```

Security Onion, Graylog, and other approved receivers must not become dependencies for local journal commit, FW capture, forwarding, or Hunter handoff.

# Net-Hunter Rules

Net-Hunter owns historical Stronghold responsibilities:

```text
receive / verify / durably commit
preserve authoritative PCAP and FW journals
process / reprocess
correlate
hunt / query
export
retention / holds / archive
FW configuration backup
future isolated IDS/inspection analysis
Pathfinder retrospective enrichment
```

Host owns FreeBSD, raw storage/HBA, ZFS, host networking, jail lifecycle, updates, and key authority.

Defined jails:

```text
1. PCAP Data Ingest Jail
2. Record Processing Jail
3. External User Interface Jail
4. FW Configuration Backup Jail
5. IDS / Inspection Jail [future / optional]
```

The External UI remains read-only toward authoritative history.

Derived indexes/correlations are rebuildable and must not rewrite authoritative PCAP/source journals.

Preserve:

```text
SEGMENT CATALOG
    where are the packets?

TRAFFIC / RECORD CATALOG
    what happened?
```

Do not require row-per-packet database storage without measurement and explicit approval.

Unknown/unsupported/malformed traffic remains discoverable where lower-level observation metadata permits.

An incomplete/rebuilding index must expose incomplete coverage rather than return an unqualified `no results` conclusion.

Reprocessing creates new/superseding derived interpretation without changing source PCAP/journals or implying later knowledge existed at original observation time.

Pathfinder enrichment is external derived context with explicit Pathfinder Record/version/provenance lineage.

New Pathfinder intelligence may trigger historical matching/reprocessing, but:

```text
retrospective match
!=
historical real-time detection
```

# Stronghold Access Rules

Stronghold Access is the third Stronghold infrastructure component when deployed and runs on a supported customer VM or bare-metal server.

Implementation belongs under:

```text
go/access/
```

Access owns:

```text
identity inputs
802.1X / network-admission coordination
AAA integration
device trust / posture
Network Access Sessions
Stronghold Access Sessions
resource authorization
policy evaluation
revocation / reevaluation
Agent policy/session distribution
controlled FW integration
optional Pathfinder risk/intelligence inputs
```

Access does not own FW packet capture/routing/NAT/network enforcement, Agent WFP implementation, Hunter authoritative history, or Pathfinder threat-intelligence authority.

Preserve three session types:

```text
Network Access Session
Stronghold Access Session
Transport Session
```

Do not collapse them into one generic session model.

802.1X is first-class. EAP-TLS is a high-assurance direction where supported. MAB is not equivalent to authenticated 802.1X.

External AAA facts are inputs to Access policy, not policy bypasses.

## Access / Agent Policy Flow

```text
FW / network admission establishes network-side context
        ↓
Stronghold Access correlates identity + posture + context
        ↓
optional approved Pathfinder intelligence contributes input
        ↓
Stronghold Access evaluates policy
        ↓
signed monotonic Agent policy generation
        ↓
Stronghold Agent verifies / validates / programs WFP
```

Do not use ad hoc per-packet FW commands as the endpoint policy model.

Access GRANT never forces FW ALLOW.

Pathfinder risk never directly forces Access REVOKE. Access policy makes the actual decision under a known generation.

# Stronghold Agent Rules

Stronghold Agent is the endpoint-side Stronghold component and endpoint PEP.

Implementation belongs under:

```text
go/agent/
```

Windows is the initial platform direction.

Use appropriate native Windows APIs/facilities. Do not repeatedly parse PowerShell, `netsh`, WMI CLI, or shell output when a suitable native API exists.

Initial endpoint enforcement direction:

```text
Go Windows service
    ↓
Windows Filtering Platform
    ↓
process/application-aware endpoint PEP
```

The Agent may stop clearly unauthorized connections before they enter the LAN, WireGuard, or FW.

```text
endpoint DENY != FW observed DENY
endpoint ALLOW != FW ALLOW
```

The first Agent implementation is process/application-aware, not a general deep-payload DPI engine.

Do not introduce a kernel-mode WFP callout driver, local TLS proxy, or deep-L7 engine merely for marketing terminology. Any such later work requires separate qualification.

Agent-reported VLAN/zone is not authoritative network-side context.

The Agent should normally not communicate directly with Pathfinder. It receives authorized policy through Stronghold Access.

Protected Endpoint full-tunnel claims require explicit testing of IPv4/IPv6, DNS, local routes, secondary NICs, virtual adapters, boot/sleep/network transitions, other VPN software, local-link protocols, transport failure, Agent failure, and local-admin tampering.

# Secure Access Rules

Secure access remains architecture-defined but implementation-deferred.

## Remote / ZTNA

Use the NIST SP 800-207 PE/PA/PEP direction:

```text
Stronghold Access
    PE / PA

Stronghold Agent
    endpoint PEP

Stronghold FW
    network PEP

WireGuard
    secure transport/data plane
```

WireGuard transport identity does not become Stronghold authorization.

Grant/deny/revoke are first-class lifecycle outcomes.

## Site-to-Site / Branch Office

Direction:

```text
L2TPv3
    ↓
IPsec protection
    ↓
site/tunnel identity
    ↓
Stronghold bridge/VLAN/routing
    ↓
normal FW policy
```

Tunnel establishment never grants unrestricted transit.

# Pathfinder Integration Rules

`docs/PATHFINDER-INTEGRATION.md` governs Stronghold↔Pathfinder behavior.

Pathfinder is a separate ISS threat-intelligence authority.

> **Pathfinder owns the organization's record and interpretation of threat intelligence. Stronghold owns observation, access/enforcement decisions, operational history, and the actions performed in the Stronghold environment.**

Stronghold may use Pathfinder for:

```text
FW operator/observable enrichment
future explicit policy inputs
IDS/IPS contextual enrichment
Net-Hunter retrospective matching/reprocessing
Stronghold Access risk/trust reevaluation
qualified Stronghold-to-Pathfinder observable submissions
```

Preserve:

```text
Pathfinder record exists            != observable malicious
Pathfinder malicious classification != compromise proven
Pathfinder confidence HIGH          != Stronghold action authorized
Pathfinder match                    != IDS detection
IDS finding                         != Pathfinder confirmation
Pathfinder risk signal              != Access Session revoked
Stronghold observation              != Pathfinder intelligence
Stronghold submission               != malicious classification
Pathfinder unavailable              != observable trusted
retrospective Pathfinder match      != historical real-time detection
```

When Pathfinder materially influences a Stronghold action, preserve the Pathfinder Record/version/provenance used and the Stronghold policy generation/action that resulted.

Pathfinder credentials are purpose-separated from HISTORY, HA, journal signing, management, IDS/inspection, WireGuard, and site-tunnel credentials.

Pathfinder is not a mandatory runtime dependency for basic capture, ordinary FW forwarding, journal commit, Hunter ingest, valid Agent WFP enforcement, or ordinary Access operation unless an explicit customer policy requires fresh intelligence.

# IDS / TLS Inspection Rules

IDS/IPS implementation remains deferred.

Future selective TLS/application inspection:

```text
client TLS
    ↓
Stronghold FW proxy
    ↓
server TLS

volatile plaintext on FW
    ↓
encrypted authenticated INSPECTION transport
    ↓
Hunter IDS / Inspection Jail
```

Stronghold FW never intentionally persists decrypted inspection payload at rest.

HISTORY and INSPECTION identities, queues, health, and failure semantics remain separate.

Findings-only is the default persistence direction; full decrypted sessions and TLS secrets are not retained by default.

Pinned and mTLS applications require explicit handling rather than generic bypass defeat.

Pathfinder complements IDS/IPS with external intelligence context but does not replace the detector or create enforcement authority.

```text
IDS finding + Pathfinder match
!=
automatic IPS action
```

Any IPS action remains an explicit Stronghold policy/enforcement result.

# Management / Configuration / Simulation

CLI, API, and future UI are clients of one Stronghold management authority.

No surface receives a separate configuration truth or bypass around candidate validation, authorization, Git-backed lineage, mandatory change attribution, activation, reconciliation, or journaling.

```text
RUNNING
   ↓
CANDIDATE
   ↓
VALIDATE
   ↓
SHOW DIFF
   ↓
DRY RUN / SIMULATE
   ↓
EXPECTED IMPACT
   ↓
MANDATORY CHANGE COMMENT
   ↓
AUTHORIZATION / APPROVAL
   ↓
GIT-BACKED FINALIZATION
   ↓
NEW STRONGHOLD GENERATION
   ↓
ACTIVATE
   ↓
RUNTIME RECONCILIATION
   ↓
POST-COMMIT OBSERVATION
```

Dry Run is non-mutating and remains prediction, not proof of actual behavior.

Rollback creates a new forward-moving generation; it never erases the mistaken one.

Native Linux/FreeBSD/WireGuard/L2TPv3/IPsec/WFP state is implementation state below Stronghold. Material native divergence becomes drift/mismatch, not silently imported Stronghold configuration.

Stable object identities survive rename/reorder where applicable.

# Stateful Enforcement

Existing state does not outrank current authorization.

Policy changes reconcile affected live sessions against the current generation.

Potential outcomes:

```text
REBOUND
TERMINATE
RESTART_REQUIRED
```

Rule movement is enforcement behavior, not cosmetic metadata.

Simulation predictions remain distinct from actual reconciliation outcomes.

# Observability

Health is domain-specific.

Keep capture, PCAP durability, forwarding, policy, routing, WAN, HA, Hunter/history transfer, journals, time, trust, storage, hardware, updates, Access, Agent, secure access, IDS/inspection, and Pathfinder integration independently observable.

A running process or interface UP does not prove subsystem health.

Alert acknowledgement/suppression/resolution never rewrites journal history or erases a gap.

Metrics measure; journals preserve meaningful authoritative transitions/actions.

# HA

Initial FW HA is active/standby only.

Each node retains its own Appliance ID, physical capture provenance, management/history/HA identities, storage, certificates, and local keys. The pair has a Cluster ID and cluster-owned virtual forwarding identity.

Virtual forwarding identity never replaces physical capture provenance.

Heartbeat/control and bulk sync are separate traffic classes.

> **Loss of peer communication is not, by itself, proof that the peer is dead.**

> **A standby must not claim ACTIVE cluster identity until the fencing/election contract permits promotion.**

Net-Hunter is not HA quorum/witness.

Secure-access/tunnel continuity requires separate qualification from ordinary forwarding failover.

# Updates / Recovery / Crypto / Platform Trust

Stronghold FW is an appliance built on Arch Linux, not a supported unrestricted customer rolling Arch host.

Qualified releases bind relevant Stronghold software, kernel, NIC driver/firmware, capture stack, nftables/netfilter, schemas, and HA compatibility.

Install/boot success is not update validation.

HA update sequence is standby-first and stops on standby update/validation failure.

Separate rebuildable platform, recoverable configuration, appliance/trust identity, authoritative history, derived state, and transient runtime state.

Configuration backup is not identity/private-key backup.

Do not restore stale runtime session/NAT state from ordinary backup.

ZFS RAID/snapshots are not DR.

Purpose-separate packet-history, journal/config, secret/private-key, derived/temp, and inspection encryption domains.

TPM/HSM may protect normal key use but must not be the sole recovery path for authoritative history.

Secure Boot, measured boot, and attestation are separate capabilities.

Development, qualification, and supported production hardware are separate categories.

# Hardware Qualification

Stronghold claims apply to qualified appliance profiles, not arbitrary hardware that boots.

Qualification binds release, CPU, RAM/ECC requirements, NIC/driver/firmware, AF_XDP mode, queue/RSS/UMEM configuration, PCIe/NUMA topology, storage, platform trust, and enabled feature profile.

Performance testing includes bandwidth and PPS, representative packet sizes, sustained duration, traffic mix, flow/session behavior, and 64-byte packet stress.

Capture-only performance is not reused as a full-feature firewall claim.

# Support / Diagnostics

Prefer structured diagnostics before native engineering access.

Stronghold has no permanent vendor support account, hidden SSH key, hidden VPN, persistent reverse shell, or automatic remote-support tunnel.

Support bundles are manifest-driven and sensitivity-aware. PCAP, secrets, private keys, decrypted inspection payload, and crash/core/memory content are never silently included.

Creating a support bundle is distinct from transmitting it.

Future remote support, if implemented, is customer initiated, time bounded, scoped, visible, attributable, and revocable.

BMC/OOB administration and Stronghold administration remain separate trust domains.

# Implementation Scope Discipline

Implementation begins with **Phase 0 — Traffic Observation Foundation** using AF_XDP.

Phase 0 includes:

```text
0.1 Capture Segment Contract
0.2 Segment and Traffic Catalog Contracts
0.3 PCAPNG Metadata Contract
0.4 FW Local Storage Contract
0.5 Capture Resource-Protection Contract
0.6 Single-Interface AF_XDP Capture Prototype
0.7 Multi-Interface / Multi-Queue AF_XDP Capture
0.8 Performance / Loss / Visibility / Qualification Baseline
0.9 Local Tier Movement / Compression
0.10 Verified Net-Hunter History Transfer
```

Do not pull routing/enforcement, full journals, retention administration, HA, complete updates, recovery/DR, encryption/key management, platform trust, complete Hunter records/search, full management/observability, Access, Agent, secure access, Pathfinder integration, IDS/IPS, TLS proxying, UI, dynamic routing, or other later systems into Phase 0 without explicit approval.

The architecture may define later behavior so Phase 0 does not block it; architecture existence is not implementation authorization.

# Repository Operations

**Do not perform repository writes unless explicitly authorized for the specific action/change set.**

This includes:

```text
commit
push
merge
PR creation
branch changes
ruleset/settings changes
issue creation/modification
deletions
other GitHub writes
```

Read-only inspection and local/offline work are permitted unless separately restricted.

One authorization does not carry forward to an unrelated later write.

# Engineering Completeness Test

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

Important paths should preserve enough state/history to answer:

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

For Pathfinder-enriched behavior, also answer:

```text
which Pathfinder record/interpretation was used?
what version/freshness/provenance did it have?
was the intelligence available at event time or applied retrospectively?
what Stronghold policy/action actually resulted?
```

# Review Expectations

Before proposing a change as complete, verify that applicable capture, authority, routing/NAT/WAN, identity, time, journals, retention, HA, updates, recovery, encryption, platform trust, hardware qualification, Hunter isolation/provenance/search, Access/Agent boundaries, secure access, Pathfinder integration, management, observability, support, IDS/inspection, and repository-write boundaries remain intact.

Nested `AGENTS.md` files may narrow subtree requirements but must not silently weaken repository-wide truth, authority, capture, security, or component-boundary rules.
