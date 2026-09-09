# Stronghold Observability, Health, and Alerting Architecture

## Purpose

This document defines the current architectural direction for Stronghold health state, metrics, alerts, capacity forecasting, and external monitoring integrations.

The governing principle is:

> **Stronghold health must describe the state of each responsibility independently; one healthy subsystem must never hide failure in another.**

A single green/red appliance indicator is not sufficient to represent Stronghold health.

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
```

An overall appliance summary may be derived for convenience, but the component states remain the meaningful facts.

Example:

```text
STRONGHOLD FW
    OPERATING — DEGRADED

Forwarding:       HEALTHY
Capture:          HEALTHY
PCAP Durability:  HEALTHY
Hunter Transfer:  FAILED
Local Backlog:    DEGRADED
Time:             HEALTHY
HA:               HEALTHY
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

Precise reasons and measurements accompany the state rather than creating an unbounded set of status names.

A component that Stronghold cannot measure must report `UNKNOWN`, `NOT_AVAILABLE`, or equivalent truthful state rather than being assumed healthy.

## Capture Health

Capture health is a first-class product responsibility and is not inferred from process uptime or Ethernet link state.

Stronghold should expose, as applicable:

```text
NIC receive/link state
capture-worker state
capture-ring occupancy/pressure
kernel/ring/capture drops
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
```

```text
capture worker RUNNING
!=
packets durably preserved
```

A running capture worker with material kernel drops is `DEGRADED`, not healthy merely because the process remains alive.

Forwarding health and capture health remain independent. Stronghold may be forwarding correctly while capture has failed, or capture may remain healthy while forwarding has failed.

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
```

Health may include:

```text
availability
capacity
pressure
read/write latency
filesystem/pool state
device health
encryption/key state
error state
```

Net-Hunter/ZFS health may additionally include vdev state, scrub/resilver status, checksum errors, degraded devices, and pool state.

```text
ZFS pool ONLINE
!=
Hunter storage healthy
```

A physically healthy pool near exhaustion may still be operationally critical.

## Storage Pressure

The established pressure states remain distinct from device-health state:

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
```

These have different operational meaning and severity.

Useful reporting includes amount, oldest pending time, current rate, and projected capacity where appropriate.

A generic undifferentiated backlog value is insufficient.

## Query Coverage Health

The Net-Hunter records/search architecture requires explicit coverage reporting.

Example:

```text
Authoritative PCAP complete through: 20:36
Record processing complete through:  20:31
Search index complete through:       20:29
```

A query extending beyond indexed coverage must disclose incompleteness rather than returning an unqualified `no results` conclusion.

```text
no indexed result
!=
no packet history
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
```

A chain mismatch or continuity failure is distinct from ordinary checkpoint/transfer lag.

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
HA peer certificate state
trust-chain validation
revocation state
explicit peer authorization
LDAPS certificate validity
RADIUS/TACACS+ reachability
journal signing credential state
recovery authority state
```

Certificate validity and peer authorization remain separate.

Credential expiration should support progressive warning thresholds rather than appearing only at expiration. Exact defaults are not yet frozen.

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

A healthy heartbeat with lagging state synchronization may produce an overall `DEGRADED` HA state.

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
policy activation result
object/FQDN resolution state
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

A fault may remain active for a long period without creating endless independent alerts.

Conceptual alert states may include:

```text
OPEN
ACKNOWLEDGED
RESOLVED
SUPPRESSED / MAINTENANCE where later justified
```

Exact vocabulary remains implementation work.

Important separations:

```text
alert acknowledged
!=
problem fixed
```

```text
alert resolved
!=
historical gap erased
```

Alert state is derived operational management state. Journals preserve the underlying event/action history.

## Maintenance and Suppression

Planned maintenance may suppress or downgrade notifications, but it does not rewrite the actual health state and does not suppress authoritative journals.

Example:

```text
FAILED — EXPECTED DURING MAINTENANCE
```

is preferable to reporting a failed subsystem as healthy.

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

CPU, latency, queue depth, ring occupancy, and similar measurements need not create permanent journal entries every second.

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
Projected WARM capacity: 11h 24m at current ingest rate
```

Capacity forecasting is more useful than a raw percentage alone and should be exposed where reliable input rates permit it.

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

## External Syslog

Syslog is an external notification/export channel, not Stronghold journal authority.

```text
syslog delivered
!=
Stronghold journal committed
```

External syslog failure does not stop internal journal advancement. Delivery failure/backlog is independently observable where implemented.

## SNMP

Stronghold should support secure network-operations monitoring. Initial security direction is **SNMPv3** for authenticated/encrypted management rather than new designs based on v1/v2c community strings.

Potential data includes interfaces, capture/drop health, storage, Hunter backlog, HA, WAN/path health, platform/hardware state, clock, and related appliance health.

SNMP is a view of Stronghold state rather than a competing source of truth.

Traps/informs should consume the same internal alert/state model as other notification adapters.

## Structured API / Future Webhooks

The management API should expose structured health and alert state. Future webhooks may support SIEM/SOAR/ticketing/automation integrations.

External receiver failure must never erase internal state or journal history.

## Telemetry Boundary

Stronghold should not silently transmit PCAP, configuration, journals, or support telemetry to Iron Signal Systems.

> **No automatic external packet/configuration telemetry to Iron Signal Systems.**

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

> **Forwarding, capture, durability, Hunter transfer, journals, time, trust, HA, storage, and hardware remain independently observable states.**

> **External monitoring and notification systems consume Stronghold state; they are not the authority for that state.**

> **Alert acknowledgement, suppression, or resolution does not rewrite the operational history that caused the alert.**

> **Metrics and journals serve different purposes and should not be collapsed into one unbounded event store.**

> **A component that cannot be measured is reported as unknown/not-available rather than assumed healthy.**

> **A resolved incident does not erase a historical capture, journal, or durability gap.**

## Truth Separations

```text
process running          != subsystem healthy
interface UP             != capture healthy
capture worker running   != PCAP durable
ZFS ONLINE               != capacity healthy
alert acknowledged       != fault resolved
alert suppressed         != event not recorded
alert resolved           != historical gap erased
syslog delivered         != journal committed
metric expired           != event never happened
monitoring unavailable   != system healthy
```

## Scope

This document defines architecture only. It does not pull a complete metrics store, alert engine, SNMP implementation, syslog exporter, webhook system, or observability UI into Phase 0.