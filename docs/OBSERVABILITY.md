# Stronghold Observability, Health, and Alerting Architecture

## Purpose

This document defines the architectural direction for Stronghold platform health, metrics, alerts, capacity forecasting, and external monitoring integrations.

Stronghold is one tightly coupled platform. Health must still remain decomposed by responsibility and component so one successful subsystem cannot conceal failure in another.

The governing principle is:

> **Stronghold health must describe the state of each responsibility independently; one healthy subsystem must never hide failure in another.**

A single green/red platform or appliance indicator is not sufficient.

## Health Domains

Stronghold should expose independent operational domains such as:

```text
PLATFORM
CAPTURE
PCAP DURABILITY
FORWARDING
POLICY
ROUTING
WAN
HA
HISTORY TRANSFER
HUNTER
JOURNALS
TIME
IDENTITY / TRUST
STORAGE
HARDWARE
UPDATE / RELEASE STATE
STRONGHOLD ACCESS
AGENT / ENDPOINT PEP
SECURE ACCESS
IDS / INSPECTION
PATHFINDER INTEGRATION
```

An overall platform/appliance summary may be derived for convenience, but the component states remain the meaningful facts.

Example:

```text
STRONGHOLD FW
    OPERATING — DEGRADED

Forwarding:          HEALTHY
Capture:             HEALTHY
PCAP Durability:     HEALTHY
Hunter Transfer:     FAILED
Local Backlog:       DEGRADED
Time:                HEALTHY
HA:                  HEALTHY
Pathfinder Context:  STALE
```

The summary must not hide which responsibility is degraded or failed.

## Health State Model

Initial conceptual component states:

```text
HEALTHY
DEGRADED
FAILED
UNKNOWN
RECOVERY
```

Subsystem-specific state may additionally expose concepts such as `STALE`, `HOLDOVER`, `NOT_AVAILABLE`, or `PARTIAL` where those terms carry real operational meaning.

Precise reasons and measurements accompany the state rather than creating an unbounded set of status names.

A component that Stronghold cannot measure must report `UNKNOWN`, `NOT_AVAILABLE`, or equivalent truthful state rather than being assumed healthy.

## Capture Health

Capture health is a first-class Stronghold FW responsibility and is not inferred from process uptime or Ethernet link state.

Stronghold should expose, as applicable:

```text
NIC receive/link state
AF_XDP/XDP operating mode
AF_XDP socket/queue state
RX/FILL/COMPLETION ring occupancy and starvation
UMEM pressure
capture-worker state
NIC/XDP/AF_XDP/user-space drops where measurable
PCAP writer state
segment-finalization state
storage write latency/queue pressure
capture storage capacity/pressure
catalog lag
```

```text
interface UP
!=
capture healthy

capture worker RUNNING
!=
packets durably preserved
```

A running capture worker with material drops is `DEGRADED`, not healthy merely because the process remains alive.

Forwarding health and capture health remain independent.

## PCAP Durability Health

PCAP durability should independently expose whether observed data is becoming durable history.

Potential state includes:

```text
active writer health
last durable segment
segment-finalization backlog
fsync/durability failures
HOT storage latency
WARM/backlog movement
oldest not-yet-durable range
capture gap state
```

```text
packet received
!=
packet durably preserved
```

## Storage Health

Storage health is layered by role/tier.

Stronghold FW examples:

```text
OS SSD
HOT NVMe
WARM/BACKLOG SSD
```

Net-Hunter examples:

```text
boot/system
FAST
WARM
HISTORY
inspection/derived datasets
```

Health may include availability, capacity, pressure, read/write latency, filesystem/pool state, device health, encryption/key state, and error state.

Net-Hunter/ZFS health may additionally include vdev state, scrub/resilver status, checksum errors, degraded devices, and pool state.

```text
ZFS pool ONLINE
!=
Hunter storage healthy
```

A physically healthy pool near exhaustion may still be operationally critical.

## Storage Pressure

The established FW pressure states remain distinct from device-health state:

```text
NORMAL
HIGH
URGENT
CRITICAL
```

Example:

```text
HOT storage device health: HEALTHY
HOT storage pressure:       CRITICAL
```

Storage pressure does not create hidden destruction authority.

## Hunter Backlog Dimensions

Stronghold must distinguish:

```text
FW TRANSFER BACKLOG
INGEST BACKLOG
PROCESSING BACKLOG
REPROCESSING BACKLOG
INDEX REBUILD BACKLOG
PATHFINDER ENRICHMENT / SYNC BACKLOG
IDS / INSPECTION PROCESSING BACKLOG
```

These have different operational meaning and severity.

Useful reporting includes amount, oldest pending time, current rate, and projected capacity where appropriate.

A generic undifferentiated backlog value is insufficient.

## Query Coverage Health

The Net-Hunter records/search architecture requires explicit coverage reporting.

Example:

```text
Authoritative PCAP complete through:       20:36
Record processing complete through:        20:31
Search index complete through:             20:29
Pathfinder enrichment complete through:    20:24
```

A query extending beyond indexed/enriched coverage must disclose incompleteness rather than returning an unqualified `no results` or unqualified threat-intelligence conclusion.

```text
no indexed result
!=
no packet history

no Pathfinder match in stale coverage
!=
observable known-safe
```

## Journal Health

Each journal can expose, as applicable:

```text
append state
last durably committed sequence
checkpoint state/checkpoint lag
transfer/anchor state
verification state
continuity state
signing-key state
external export state
```

A chain mismatch or continuity failure is distinct from ordinary checkpoint/transfer/export lag.

External SIEM delivery failure does not imply local journal failure.

```text
SIEM export failed
!=
Stronghold journal failed
```

## Time Health

The established clock states remain directly observable:

```text
SYNCHRONIZED
HOLDOVER
UNSYNCHRONIZED
CLOCK_FAULT
```

Useful detail may include selected time source, offset, jitter, last successful synchronization, holdover duration, and clock-correction events.

Timestamp precision must not be displayed as proof of timestamp accuracy.

## Identity and Trust Health

Stronghold should expose important identity/trust conditions, including where applicable:

```text
appliance certificate validity/expiration
Hunter peer certificate state
Access peer certificate state
Agent enrollment/certificate state
HA peer certificate state
Pathfinder integration credential state
trust-chain validation
revocation state
explicit peer authorization
LDAPS certificate validity
RADIUS/TACACS+ reachability
journal signing credential state
recovery authority state
```

Certificate validity and peer authorization remain separate.

Credential expiration should support progressive warning thresholds rather than appearing only at expiration.

## Stronghold Access Health

Stronghold Access is the third Stronghold infrastructure component when deployed and requires its own health domain.

Potential dimensions include:

```text
Access service availability
Policy Engine state
Policy Administrator state
identity-provider availability
MFA dependency state
AAA/RADIUS dependency state
802.1X / network-admission integration state
Access Session state/count
policy generation state
FW control-contract state
Agent policy-distribution state
revocation processing
CoA / reauthentication integration
Pathfinder input state where configured
control-plane database/storage health
```

A running Access service does not prove authorization processing is healthy.

```text
Access process running
!=
Policy Engine healthy

AAA reachable
!=
Access authorization healthy
```

## Stronghold Agent / Endpoint PEP Health

Agent health must distinguish service process state from effective endpoint enforcement.

Potential dimensions include:

```text
Agent service state
endpoint identity/enrollment state
current policy generation
policy signature/validation state
policy lease / holdover / expiry
WFP provider/filter state
endpoint PEP verification state
control-channel state
Stronghold Access reachability
WireGuard/Protected Endpoint transport where configured
posture collection state where configured
endpoint decision-record delivery/backlog
local tamper/drift indicators where measurable
```

Mandatory separations:

```text
Agent service running
!=
endpoint PEP healthy

policy current
!=
WFP enforcement healthy

Access reachable
!=
policy current
```

## Secure Access Health

Secure access is not represented as a generic `VPN_UP` bit.

### Zero-Trust Remote Access

Potential dimensions include:

```text
Stronghold Access PE/PA state
Agent endpoint PEP state
FW network PEP state
identity/MFA/posture dependencies
Access Session state
WireGuard listener/peer/session state
revocation processing
policy lease / holdover state
```

```text
WireGuard tunnel up
!=
resource authorized
```

### Site-to-Site / Branch Office

Potential dimensions include:

```text
L2TPv3 tunnel state
IPsec SA state
peer authentication
rekey state
endpoint/WAN reachability
bridge/VLAN/route integration
packet/drop/error state
```

```text
IPsec SA healthy
!=
application path healthy
```

## IDS / Inspection Health

IDS/inspection health remains separate from capture and forwarding health.

Potential dimensions include:

```text
proxy availability
inspection-policy activation
inspection certificate/trust state
origin-certificate validation state
INSPECTION transport authorization
inspection feed throughput/drop/gap state
IDS jail health
processing backlog
ruleset/engine generation
inspection storage health
inspection coverage
Pathfinder enrichment availability for IDS correlation
```

Healthy forwarding must not hide degraded IDS coverage, and degraded IDS must not be reported as packet-observation loss when authoritative capture remained healthy.

```text
IDS healthy
!=
Pathfinder healthy

IDS unavailable
!=
authoritative capture unavailable
```

## Pathfinder Integration Health

Pathfinder integration is an independent external-intelligence health domain.

Stronghold should expose, where applicable:

```text
integration enabled/disabled state
Pathfinder endpoint reachability
authentication / explicit peer authorization
TLS / trust state
last successful synchronization
current intelligence generation/version
last accepted interpretation update
intelligence freshness / age
sync backlog count/bytes where applicable
oldest pending enrichment
rate-limit/backpressure state
FW local intelligence-cache state where used
Access intelligence-input freshness
Hunter retrospective-enrichment progress
submission/output backlog where Stronghold-to-Pathfinder feedback is enabled
```

Reachability is not freshness.

```text
Pathfinder reachable
!=
Pathfinder intelligence current

Pathfinder authentication valid
!=
Pathfinder data fresh

Pathfinder sync complete
!=
all Stronghold policy updated

Pathfinder unavailable
!=
observable trusted
```

A Pathfinder outage must not be collapsed into IDS failure, Access failure, or packet-capture failure.

If an explicitly configured policy requires fresh Pathfinder data, the resulting policy/authorization degradation must identify that dependency rather than report a generic failure.

## HA Health

HA is not represented as a single `UP/DOWN` bit.

Useful dimensions include:

```text
peer heartbeat
config synchronization
session/state synchronization
software/protocol compatibility
platform-trust eligibility
fencing readiness
peer forwarding/capture readiness
```

A healthy heartbeat with lagging state synchronization may produce a `DEGRADED` HA state.

## WAN Health

WAN/path health remains destination/service aware.

Example:

```text
WAN1 physical link:     UP
WAN1 baseline Internet: HEALTHY
Google path:            DEGRADED
Microsoft 365 path:     HEALTHY
```

One degraded probe does not automatically declare the entire WAN unusable.

## Routing Health

Stronghold should distinguish configured routes from operationally usable routes and expose causes such as next-hop resolution or installation failure.

```text
configured route
!=
usable route
```

## Policy Health

Potential policy health dimensions include:

```text
running Stronghold config generation
compiled policy generation
kernel enforcement generation
Access policy generation where applicable
Agent policy generation where applicable
policy activation result
object/FQDN resolution state
Pathfinder intelligence generation/freshness where policy consumes it
```

A generation mismatch is significant even if traffic appears to be flowing.

## Platform and Hardware Health

Platform health may include:

```text
CPU pressure
memory pressure
thermal state
service supervision
kernel/platform state
filesystem state
qualified hardware state
Secure Boot/measured-boot state when later enforced
```

Qualified production hardware may additionally expose ECC, SMART/NVMe health, HBA state, fan/PSU state, PCIe errors, NIC errors, and firmware qualification.

Unavailable sensors/counters are reported as unavailable rather than silently green.

## Health Engine vs Alert Engine

Stronghold separates continuous state from notification lifecycle.

```text
measurements / operational state
        ↓
HEALTH ENGINE
        ↓
state transition
        ↓
ALERT ENGINE
        │
        ├── CLI / UI
        ├── API
        ├── syslog
        ├── SNMP notification
        └── future webhook/email
```

Meaningful operational facts and actions remain separately journaled.

## Alert Lifecycle

Conceptual alert states may include:

```text
OPEN
ACKNOWLEDGED
RESOLVED
SUPPRESSED / MAINTENANCE where justified
```

Important separations:

```text
alert acknowledged
!=
problem fixed

alert resolved
!=
historical gap erased
```

Alert state is derived operational management state. Journals preserve the underlying event/action history.

## Maintenance and Suppression

Planned maintenance may suppress or downgrade notifications, but it does not rewrite actual health state and does not suppress authoritative journals.

```text
notification suppressed
!=
event not recorded
```

## Metrics vs Journals

Stronghold separates high-frequency measurement from authoritative event/state-transition history.

```text
METRICS
    high-frequency measurements / trends

JOURNALS
    meaningful authoritative transitions, actions, failures, and results
```

CPU, latency, queue depth, ring occupancy, intelligence age, and similar measurements need not create permanent journal entries every second.

Threshold/state transitions may create authoritative journal facts.

```text
metric retention expired
!=
failure event never happened
```

The exact metrics-store technology is not selected.

## Actionable Alerts

Alerts should explain operational impact where Stronghold can establish it.

Example:

```text
HISTORY TRANSFER DEGRADED

Hunter unreachable for: 17m 42s
Forwarding:              unaffected
Capture:                 continuing locally
Local backlog:           83 GB
Oldest pending:          17m 39s
Projected capacity:      11h 24m at current ingest rate
```

Pathfinder example:

```text
PATHFINDER INTELLIGENCE STALE

Pathfinder transport:         reachable
Last successful sync:         2h 17m ago
Current accepted generation:  PF-GEN-184
Latest advertised generation: PF-GEN-189
FW forwarding:                unaffected
Capture:                      unaffected
Access policies requiring
fresh intelligence:           DEGRADED
Hunter enrichment backlog:    14,281 records
```

Capacity/freshness forecasting is more useful than a raw percentage or green/red bit where reliable input exists.

## Capture-Loss Alerts

Capture/durability failures deserve explicit first-class events such as conceptual:

```text
CAPTURE_DROP_DETECTED
CAPTURE_RING_PRESSURE
PCAP_WRITE_FAILED
SEGMENT_FINALIZATION_FAILED
STORAGE_EXHAUSTED
```

Where known, alerts/history preserve affected interface, first/last affected time, drop count, reason/source, forwarding impact, and whether a capture/history gap exists.

When the fault resolves, the alert may resolve; the historical gap remains.

## External Syslog / SIEM Export

Syslog and structured external log delivery are notification/export channels, not Stronghold journal authority.

```text
syslog delivered
!=
Stronghold journal committed
```

External delivery failure does not stop internal journal advancement. Delivery failure/backlog is independently observable where implemented.

## SNMP

Stronghold should support secure network-operations monitoring. Initial security direction is **SNMPv3** for authenticated/encrypted management rather than new designs based on v1/v2c community strings.

Potential data includes interfaces, capture/drop health, storage, Hunter backlog, Access/Agent health, secure access, Pathfinder freshness, HA, WAN/path health, platform/hardware state, clock, and related health.

SNMP is a view of Stronghold state rather than a competing source of truth.

## Structured API / Future Webhooks

The management API should expose structured health and alert state. Future webhooks may support SIEM/SOAR/ticketing/automation integrations.

External receiver failure must never erase internal state or journal history.

## Telemetry Boundary

Stronghold should not silently transmit PCAP, configuration, journals, endpoint information, Pathfinder-derived data, or support telemetry to Iron Signal Systems.

> **No automatic external packet/configuration/support telemetry to Iron Signal Systems.**

Stronghold-to-Pathfinder observable submission, if enabled, is a separately configured integration governed by `docs/PATHFINDER-INTEGRATION.md`; it is not hidden product telemetry.

Any future remote-support/telemetry feature requires explicit operator-visible scope and authorization.

## Diagnostic Logs

Stronghold may maintain bounded ordinary diagnostic logs for implementation troubleshooting.

```text
JOURNAL
    durable authoritative operational history

DIAGNOSTIC LOG
    implementation troubleshooting data
```

Diagnostic logs are lower-authority, bounded/rotated, and must not be represented as the sole record of important Stronghold operational facts.

```text
debug log lost
!=
authoritative journal lost
```

## Observability Invariants

> **Forwarding, capture, durability, Hunter transfer, journals, time, trust, HA, storage, hardware, Access, Agent, secure access, IDS/inspection, and Pathfinder integration remain independently observable states.**

> **External monitoring and notification systems consume Stronghold state; they are not the authority for that state.**

> **Pathfinder reachability does not prove intelligence freshness, and Pathfinder unavailability does not mean an observable is trusted.**

> **Alert acknowledgement, suppression, or resolution does not rewrite the operational history that caused the alert.**

> **Metrics and journals serve different purposes and should not be collapsed into one unbounded event store.**

> **A component that cannot be measured is reported as unknown/not-available rather than assumed healthy.**

> **A resolved incident does not erase a historical capture, journal, durability, inspection, authorization, or intelligence-coverage gap.**

## Truth Separations

```text
process running             != subsystem healthy
interface UP                != capture healthy
capture worker running      != PCAP durable
ZFS ONLINE                  != capacity healthy
alert acknowledged          != fault resolved
alert suppressed            != event not recorded
alert resolved              != historical gap erased
syslog delivered            != journal committed
metric expired              != event never happened
monitoring unavailable      != system healthy
Access process running      != Access authorization healthy
Agent service running       != endpoint PEP healthy
policy current              != endpoint enforcement healthy
WireGuard tunnel up         != resource authorized
IPsec SA up                 != application path healthy
IDS healthy                 != Pathfinder healthy
Pathfinder reachable        != intelligence current
Pathfinder sync complete    != all Stronghold policy updated
Pathfinder unavailable      != observable trusted
```

## Scope

This document defines architecture only.

It does not pull a complete metrics store, alert engine, SNMP implementation, syslog exporter, webhook system, Access/Agent observability implementation, Pathfinder integration, or observability UI into Phase 0.
