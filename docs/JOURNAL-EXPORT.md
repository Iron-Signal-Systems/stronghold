# Stronghold Journal Export and SIEM Integration Architecture

## Purpose

This document defines the current architecture for exporting Stronghold journal information to external SIEM and log-ingestion systems while preserving Stronghold's own journal authority and Net-Hunter history semantics.

The governing principle is:

> **Stronghold journals remain the authoritative record of what Stronghold did. External SIEM and log platforms receive structured exported copies for operations, correlation, alerting, and investigation; they do not become Stronghold's journal authority merely because they received an event.**

Stronghold FW must remain useful as a standalone firewall. Net-Hunter is optional for firewall operation, while adding long-term packet/history, deeper hunt/reprocessing, configuration backup, and other historical capabilities. Journal export to external systems is independent of whether Net-Hunter is present.

## Destination Model

Stronghold may deliver journal-derived events to multiple independent destinations.

Conceptually:

```text
STRONGHOLD FW JOURNALS
        │
        ├──► local recent operational view
        │
        ├──► Net-Hunter history transfer
        │
        ├──► SIEM / Security Onion
        │
        ├──► log ingestion / Graylog
        │
        └──► other approved structured receivers
```

A deployment may use:

```text
FW only
FW + external SIEM
FW + Net-Hunter
FW + Net-Hunter + external SIEM
```

None of these combinations changes the firewall's core forwarding/capture authority.

## Journal Domains

The established Stronghold journal domains remain independent:

```text
Administrative
System / Health
Traffic Decision
Trust / Identity
Time / Clock
Hunter Processing
```

A standalone FW produces the applicable FW-origin journal domains. Hunter-origin processing records remain Hunter records.

External export does not collapse the journal domains into one generic source internally. An adapter may map fields to a destination schema, but the original Stronghold journal domain and source identity remain represented.

## Export Record Identity

A journal export should preserve enough stable identity for an external platform to correlate the event back to Stronghold history.

Where applicable, exported records should include or reference:

```text
Stronghold schema/version
Journal ID
journal domain
origin Appliance ID
Cluster ID when applicable
journal epoch
journal sequence
operation/correlation ID
configuration generation
Stronghold event/action/result type
UTC event time
clock-confidence/time state where relevant
source management/interface context where relevant
policy/route/NAT/WAN identifiers where relevant
result / failure / NOT_PERFORMED state
```

Where the journal-integrity profile later permits, an export may also include source-entry hash/checkpoint/segment references useful for verification and correlation.

An external representation may add destination-specific metadata, but it must not silently rewrite Stronghold meaning.

## Structured Export

The preferred architecture is structured export rather than free-form human text as the only representation.

Stronghold may support adapters for common SIEM/log-ingestion mechanisms such as secure syslog and structured API/event transports. Exact protocols and schemas remain to be frozen during implementation qualification.

Security Onion, Graylog, and similar systems are intended examples of external consumers, not privileged Stronghold dependencies.

Human-readable summaries may accompany structured fields, but automation must not be forced to parse prose to recover basic Stronghold facts.

## Secure Transport Direction

Production journal export should use authenticated/encrypted transport where the destination technology permits it.

Stronghold should not design its primary modern integration around unauthenticated plaintext UDP logging.

Exact transport profiles remain to be selected, but candidate directions may include:

```text
syslog over TLS
mutually authenticated TLS where supported
structured HTTPS/API delivery
other explicitly approved secure adapters
```

Destination credentials/secrets remain Stronghold secret references rather than plaintext configuration fields.

## Export Independence

Journal advancement is independent from external export availability.

```text
journal committed locally
!=
SIEM event delivered
```

If Security Onion, Graylog, or another destination is unavailable:

```text
Stronghold journal continues
capture continues
forwarding continues
local operational view continues
Net-Hunter transfer continues if available
external-export state becomes DEGRADED/FAILED
```

External export must not become a synchronous dependency for packet forwarding, capture, or authoritative journal commit.

## Export Backlog

Stronghold may maintain a bounded journal-export queue/backlog so temporary receiver outages do not immediately lose external visibility.

The queue is secondary work and must not be permitted to starve:

```text
packet acquisition
active PCAP writes
essential forwarding/enforcement
authoritative journal commit
critical HA control
authoritative history protection
```

Backlog should expose, where practical:

```text
destination
state
queued events/bytes
oldest pending event
last successful delivery
retry state
coverage / dropped-export interval if loss occurs
```

If the export queue cannot preserve every external copy, Stronghold reports the export gap truthfully. It does not sacrifice the authoritative local journal to preserve an external projection.

```text
SIEM export gap
!=
Stronghold journal gap
```

## Delivery Acknowledgement Semantics

Net-Hunter and external SIEM delivery have different authority.

Net-Hunter history transfer uses the Stronghold durable-history contract:

```text
receive
verify
durably commit
ACK
```

That ACK can participate in Stronghold's local history/backlog lifecycle because Hunter is a Stronghold history authority.

External SIEM/log receiver delivery is different:

```text
SIEM accepted event
!=
Net-Hunter verified durable-history ACK
```

A syslog/TLS/API success from an external receiver does not grant permission to delete otherwise protected Stronghold history and does not prove the receiver preserved Stronghold integrity/retention semantics.

## Filtering and Routing

Administrators may need to choose which journal domains/events are exported to which destinations.

Conceptually:

```text
Administrative      → SIEM A
System / Health     → SIEM A + NOC logging
Traffic Decision    → SIEM A / Security Onion
Trust / Identity    → SIEM A
Time / Clock        → SIEM A
```

Exact filtering syntax remains to be frozen.

Filtering controls the exported copy only.

```text
not exported
!=
not journaled
```

Stronghold must not stop recording an authoritative journal event merely because an administrator chose not to send that event class externally.

## Traffic Decision Export

Traffic Decision Journal export is particularly useful for external security correlation.

Where applicable, exported decision records may include:

```text
source/destination tuple
original and translated tuple
physical/logical interface
VLAN / zone
Policy ID and rule position
configuration generation
route / next-hop reference
WAN selection
allow / drop / reject
actual forwarding result
NOT_PERFORMED stages
failure reason
operation / flow correlation references
```

External platforms can correlate these records with endpoint, identity, DNS, IDS, and other security information without Stronghold pretending that the SIEM copy is authoritative packet history.

## PCAP Boundary

Journal export does not automatically export PCAP.

```text
journal export enabled
!=
packet export enabled
```

PCAP/history export remains a separate explicit capability with its own authorization, provenance, bandwidth, retention, and sensitivity controls.

An external SIEM receiving a Traffic Decision record must not be assumed to possess the corresponding authoritative packet bytes.

## Standalone FW Behavior

Net-Hunter is not required for Stronghold FW to provide:

```text
capture
firewalling
L2/L3
NAT
multi-WAN
policy/routing
journaling
management
health/diagnostics
recent operational traffic/decision visibility
external journal export
```

The local recent working set is intended to let an administrator validate policy, route, NAT, WAN, and actual disposition in near real time without waiting for Net-Hunter.

Net-Hunter adds long-term packet/history preservation, deeper historical search/reprocessing, configuration backup, future IDS inspection persistence, and related historical services.

## Local Journal / History Window Direction

The current FW direction is a short local operational window rather than long-term historical storage.

Normal target direction:

```text
recent local working/history window: approximately 30–60 minutes
Hunter outage becomes CRITICAL:      approximately 4 hours
qualified local capacity target:     approximately 8 hours equivalent ingest
```

The exact capacity is ultimately determined by qualified aggregate observed traffic, journal rate, hardware, and safety margin rather than nominal interface speed alone.

The approximately 8-hour capacity direction provides substantial margin beyond the 4-hour critical Hunter-outage objective.

External SIEM export does not remove the need for local operational buffering or, when deployed, Net-Hunter durable-history transfer.

## Near-Real-Time Operational View

The FW recent working set should make recent journal/traffic decisions operator-visible in seconds rather than requiring Hunter processing.

This view may correlate recent traffic with:

```text
configuration generation
Policy ID / rule position
route / next hop
NAT
WAN selection
capture state
final action/result
failure / NOT_PERFORMED reason
```

The exact visibility-latency SLO must be benchmarked before being promised as a product number.

If recent-view indexing lags under load, Stronghold reports the lag/coverage explicitly rather than displaying stale information as current.

```text
recent-view lag
!=
capture loss
```

## Observability of Export

Each external destination should have independent health/coverage state.

Conceptual dimensions include:

```text
configured
authorized / TLS state
connected/reachable
last successful delivery
backlog
oldest pending
retry state
coverage gap
schema/adapter state
```

One failed destination does not make another destination failed.

## External Platform Authority

External SIEM/log systems may retain Stronghold exports longer than the FW and may become important organizational records. That does not change the Stronghold authority model.

```text
external record retained
!=
Stronghold source journal retained

external receiver accepted event
!=
Stronghold event verified by Net-Hunter

SIEM correlation
!=
Stronghold authoritative observation
```

Stronghold should expose enough stable identifiers for an investigator to correlate external events back to Stronghold history where that history remains available.

## Resource Priority

Journal export is secondary to live appliance responsibilities.

A later qualified priority model must preserve at minimum:

```text
packet acquisition
active PCAP writes
essential forwarding/enforcement
authoritative journal durability
critical HA control
history/backlog safety
    ↓
external journal export / catch-up
```

Export recovery after a receiver outage must be rate controlled so catch-up does not create new capture loss or forwarding degradation.

## Invariants

> **Stronghold journals remain authoritative Stronghold history regardless of whether Net-Hunter or an external SIEM is deployed.**

> **Journal export to Security Onion, Graylog, or another external consumer is independent of Net-Hunter and is available to a standalone Stronghold FW.**

> **External delivery failure never blocks authoritative journal commit, packet capture, or normal forwarding.**

> **An external receiver acknowledgement is not equivalent to Net-Hunter verified durable-history acknowledgement.**

> **Export filtering changes only what is sent externally; it never silently changes what Stronghold journals.**

> **Journal export does not implicitly authorize or enable PCAP export.**

> **Export backlog/catch-up is secondary and must not starve live observation.**

## Truth Separations

```text
journal committed locally       != SIEM event delivered
SIEM event delivered            != Net-Hunter history committed
not exported                    != not journaled
SIEM export gap                 != Stronghold journal gap
journal export enabled          != PCAP export enabled
external record retained        != Stronghold source retained
SIEM correlation                != authoritative packet observation
recent-view lag                 != capture loss
Net-Hunter absent               != FW journaling unavailable
Net-Hunter absent               != external journal export unavailable
```

## Scope

This document defines architecture only. It does not select the final syslog framing, JSON schema, SIEM normalization mapping, Graylog input format, Security Onion ingestion pipeline, delivery queue implementation, or external-export retention policy. Those choices must be qualified later against capture-first resource constraints and the Stronghold journal-integrity contract.
