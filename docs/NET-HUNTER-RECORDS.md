# Stronghold Net-Hunter Records, Search, and Reprocessing Architecture

## Purpose

This document defines the architectural direction for Stronghold Net-Hunter records, indexing, search, correlation, query coverage, historical reprocessing, and external intelligence enrichment.

It does not select a database product or implementation technology. It defines what any later implementation must preserve.

The governing principle is:

> **Indexes and derived records exist to locate and explain authoritative history; they are never themselves the sole proof that the history existed.**

Stronghold preserves the distinction between authoritative source history and searchable derived interpretation.

```text
AUTHORITATIVE

PCAPNG
FW source journals
Hunter Processing Journal
configuration generations
integrity / lineage information

        ↓ derive / correlate / enrich

SEARCHABLE / DERIVED

segments
flows / sessions
protocol records
address / name relationships
decision correlations
Pathfinder intelligence correlations
indexes
timelines
search facets
```

If derived state is lost, Stronghold rebuilds it from authoritative source history where that source remains available.

If authoritative PCAP or source journals are lost, Stronghold reports actual historical loss. Rebuilding an index must never be presented as recovering source history that no longer exists.

## Authority Boundaries

Stronghold Net-Hunter maintains three Stronghold truth classes:

```text
WHAT WAS PRESENTED
    authoritative packet history / PCAPNG

WHAT STRONGHOLD DID
    authoritative FW and Hunter journals

WHAT STRONGHOLD UNDERSTOOD
    derived records, correlation, enrichment, intelligence context, and indexes
```

Iron Signal Systems Pathfinder is a separate authority for threat-intelligence records and interpretation.

```text
Stronghold observation
!=
Pathfinder intelligence

Pathfinder interpretation
!=
Stronghold historical observation
```

Pathfinder-derived correlations in Hunter are derived state. They never rewrite source PCAP, FW journals, or the time at which Stronghold originally knew something.

See `docs/PATHFINDER-INTEGRATION.md`.

## Two-Catalog Architecture

Net-Hunter separates packet-location metadata from traffic interpretation.

```text
SEGMENT CATALOG
    where are the packets?

TRAFFIC / RECORD CATALOG
    what happened?
```

### Segment Catalog

The segment catalog locates authoritative packet objects and tracks operational state close to packet storage.

Conceptual segment information includes:

```text
Segment ID
Origin Appliance ID
source physical interface
source VLAN / observation context where applicable
start time
end time
packet count
byte count
drop / loss accounting
PCAP format/version
integrity hash
verification state
storage tier
storage location
retention state
hold state
```

The segment catalog must remain sufficient to narrow candidate PCAP history even when higher-level derived indexes are unavailable or incomplete.

### Traffic / Record Catalog

The traffic catalog contains derived records used to search, correlate, explain, enrich, and pivot into authoritative history.

It should not require a database row for every captured packet merely to make traffic searchable.

```text
PCAP
    authoritative packet detail

FLOW / SESSION RECORDS
    efficient traffic search

PROTOCOL / EVENT RECORDS
    higher-level derived facts

INTELLIGENCE CORRELATION
    Pathfinder or other explicitly approved external enrichment

PCAP LOCATORS
    references back toward authoritative packet history
```

## Row-Per-Packet Is Not the Primary Search Model

Stronghold does not make a row-per-packet database the primary search architecture.

At meaningful packet rates, a required database row for every Ethernet frame creates unnecessary storage, indexing, and write amplification and risks turning the derived database into the practical source of truth.

Packet-level detail remains in authoritative PCAPNG. Derived structures exist to narrow the search space and explain activity.

## Conceptual Record Families

Initial logical record families include:

```text
SEGMENT
OBSERVATION
FLOW / SESSION
L2 / CONTROL-PLANE
NAME / ADDRESS RELATIONSHIP
FIREWALL DECISION
ROUTE / WAN / NAT CORRELATION
SYSTEM / CONFIGURATION REFERENCES
ACCESS / ENDPOINT CORRELATION
PATHFINDER INTELLIGENCE CORRELATION
DERIVED ANALYSIS
PROCESSING / INDEX STATE
```

These are conceptual record families, not frozen database tables.

## Flow and Session Records

Flow/session records are expected to become one of Net-Hunter's primary search structures.

Conceptual information may include:

```text
Flow ID
Origin Appliance ID
first seen
last seen
ingress physical interface
ingress VLAN
source zone
source MAC
destination MAC
source IP
destination IP
source port
destination port
protocol
packet count
byte count
original tuple
translated tuple where applicable
selected WAN
egress interface
egress VLAN
configuration generation
policy decision references
Access Session / endpoint references where correlated
capture segment references
```

Not every field applies to every flow. Stronghold must not force non-IP traffic into an IP-flow schema merely for implementation convenience.

## Layer-2 and Control-Plane Records

Stronghold capture is not IP-centric. Net-Hunter must preserve useful derived records for observable Layer-2 and control-plane traffic where decoders exist.

Examples include:

```text
ARP
IPv6 NDP
DHCP / BOOTP
DHCPv6
CDP
LLDP
STP / RSTP / MSTP
LACP
802.1X / EAPOL
VRRP
OSPFv2 / OSPFv3
IGMP
unknown EtherTypes
```

A decoded record never replaces the source frame.

## Historical Relationships

Net-Hunter treats relationships as time-bounded observations rather than one timeless current mapping.

Examples include:

```text
MAC ↔ IP
IP ↔ hostname
hostname ↔ DNS answer
DHCP lease
VLAN membership
CDP / LLDP adjacency
user/device ↔ Access Session where source-qualified
endpoint process ↔ destination where source-qualified
```

Stronghold preserves historical intervals instead of overwriting them with the newest relationship.

## Provenance on Derived Facts

Every material derived fact should be able to answer:

```text
Where did this come from?
```

Derived provenance should identify, as applicable:

```text
source Appliance ID
source segment / journal object
source packet or packet-range locator
observation time
processing time
decoder / correlator identity
processing generation
relationship type / confidence
external enrichment source
external record / interpretation identity
```

### Direct, Correlated, External, and User Context

Conceptual provenance classes include:

```text
DIRECT
    parsed directly from source packet or authoritative journal

CORRELATED
    derived by combining direct observations

EXTERNAL
    supplied by an approved enrichment source such as Pathfinder

USER_SUPPLIED
    analyst annotation/context

NOT_KNOWN
```

A Pathfinder classification must remain `EXTERNAL`/intelligence-derived context rather than being represented as if directly observed on the wire.

## Search Philosophy

Net-Hunter search should answer operational questions rather than require operators to understand the internal database schema.

Examples include:

```text
show traffic involving 10.10.50.27
show everything from MAC aa:bb:cc:dd:ee:ff
show denied traffic from VLAN 20
show sessions that used WAN2
show traffic that matched policy ID 1042
show packets around this DHCP lease
show everything involving port 445
show traffic between 01:00 and 03:00
show events during a firewall configuration change
show traffic before and after HA failover
show traffic associated with Pathfinder Record PF-...
show prior contact with observables Pathfinder now classifies as C2
```

Search results should support pivots such as:

```text
IP / domain / observable
 ↓
flows
 ↓
policy decision
 ↓
Access / endpoint context
 ↓
NAT / WAN / route
 ↓
Pathfinder intelligence
 ↓
PCAP
```

CLI/UI/API design should hide database mechanics without hiding source and provenance.

## High-Value Search Pivots

Likely high-value pivots include:

```text
time
source / destination IP
source / destination MAC
source / destination port
protocol
VLAN
zone
physical interface
policy ID
configuration generation
WAN
NAT tuple
hostname / FQDN
segment ID
Appliance ID
Cluster ID
journal operation ID
Access Session ID
Endpoint Decision ID
Pathfinder Record ID
Pathfinder observable / classification where indexed
```

Actual indexing strategy is benchmark-driven and remains implementation work.

## Time-Range Pruning

Most investigations have a bounded time range. Net-Hunter should use segment metadata and later partition/index structures to narrow historical search early rather than scanning all retained history when unnecessary.

## Database / Index Technology Is Not Yet Selected

Stronghold does not currently bind Net-Hunter to PostgreSQL, OpenSearch, ClickHouse, SQLite, a custom index, or another specific database technology.

Any candidate implementation must be evaluated against authority boundaries, record types, ingest rate, query patterns, retention, rebuild behavior, expected scale, query latency, failure behavior, and operational complexity.

The architecture must not be reshaped merely to fit a convenient database product.

## Indexes Are Rebuildable

> **An index may be deleted and rebuilt without altering authoritative source history.**

Stronghold should distinguish:

```text
Search index: REBUILDING
Authoritative PCAP: AVAILABLE
```

from actual source-history unavailability.

## Query Coverage Is Explicit

> **Stronghold must never present an incomplete index as complete history.**

A zero-result answer from an incomplete index must not be represented as proof that no matching traffic exists.

Net-Hunter must distinguish conceptually:

```text
not observed
not captured
captured but not processed
processed but not decoded
decoded but not indexed
indexed and no result
Pathfinder enrichment pending
Pathfinder enrichment stale/unavailable
history destroyed
history unavailable
```

These states answer different operational questions and must not be collapsed.

## Search Fallback to Authoritative Catalogs

When derived indexes are unavailable or incomplete, Net-Hunter should retain a slower path through the segment catalog and authoritative source metadata.

```text
normal hunt
    ↓
derived indexes

index unavailable / incomplete
    ↓
segment catalog
    ↓
candidate PCAP objects
    ↓
targeted retrieval / reprocessing
```

The fallback may be slower, but the system should not become blind merely because a derived index is rebuilding.

## Unknown and Unsupported Traffic Remains Discoverable

Unknown traffic must not disappear from search merely because no higher-level decoder exists.

Where available from observation metadata, Hunter should preserve searchable facts such as EtherType, IP protocol number, MAC addresses, interface, VLAN, time, and frame/packet size along with explicit decoder state.

Examples:

```text
UNSUPPORTED
DECODER_NOT_AVAILABLE
PARSE_PARTIAL
MALFORMED
DECODER_ERROR
```

Parser failure or unsupported protocol is information, not permission to erase discoverability.

## Reprocessing Architecture

Authoritative PCAP/source journals allow Net-Hunter to reinterpret old history with newer decoders, correlators, and intelligence.

Reprocessing never rewrites the original observation.

```text
Observation time: T1
Original processing: DECODER_NOT_AVAILABLE
Reprocessed time: T2
New decoder/correlator/intelligence: ...
New derived fact: ...
```

Stronghold must not present a later derived fact as knowledge Stronghold possessed at the original observation time.

### Processing and Decoder Versioning

Material derived records should preserve, as applicable:

```text
source object identity
derived-record generation
decoder / correlator version
Pathfinder interpretation/version where applicable
processing time
supersession relationship
```

### Targeted Reprocessing

Net-Hunter should eventually support targeted reprocessing by bounded scope such as:

```text
time range
Appliance ID
VLAN
segment range
protocol / decoder family
Pathfinder observable / Record ID
intelligence generation/update
```

Targeted reprocessing operations belong in the Hunter Processing Journal.

### Full Reprocessing and Derived Generations

For significant decoder/schema/intelligence changes, Stronghold should prefer building a new derived generation rather than leaving partially migrated current state.

```text
Derived Generation 17
    current

Derived Generation 18
    building
        ↓
validate
        ↓
switch default query generation
        ↓
retain / expire older derived generation by policy
```

Exact generation mechanics remain implementation work.

## Pathfinder Retrospective Matching

Pathfinder integration is one of the primary uses of Net-Hunter's preserved historical truth.

New intelligence can be applied to old history without changing what was originally observed.

```text
PATHFINDER UPDATE
        ↓
observable newly classified / associated
        ↓
NET-HUNTER
        ↓
historical search / correlation / reprocessing
        ↓
prior endpoints
prior sessions / flows
prior Stronghold decisions
source PCAP references
```

Example:

```text
Pathfinder Record: PF-98471
Current interpretation: known C2 infrastructure

Net-Hunter historical result:
    first Stronghold observation: 2026-07-11T03:17:22Z
    endpoints: ...
    sessions: ...
    related PCAP: available
```

The result must state the distinction between original observation time and intelligence application time.

```text
observed in July
!=
known malicious in July

retrospective match today
!=
historical real-time detection
```

A Pathfinder update may create a new derived correlation generation. It must never rewrite old FW decisions, old IDS findings, source journals, or authoritative PCAP as though Pathfinder's current knowledge existed then.

## Processing Failures Are Journaled

If processing or Pathfinder enrichment fails after authoritative ingest succeeds, the authoritative segment remains committed.

Potential Hunter Processing Journal events include:

```text
PROCESSING_STARTED
PROCESSING_FAILED
PROCESSING_RETRY_STARTED
PROCESSING_SUCCEEDED
INDEX_BUILD_STARTED
INDEX_BUILD_FAILED
INDEX_BUILD_SUCCEEDED
REPROCESSING_STARTED
REPROCESSING_COMPLETED
PATHFINDER_ENRICHMENT_STARTED
PATHFINDER_ENRICHMENT_FAILED
PATHFINDER_ENRICHMENT_COMPLETED
```

Later success does not erase earlier failure.

## Record Processing Jail Boundary

The Record Processing Jail may read authoritative PCAP/source journals and write derived records, indexes, processing metadata, and Pathfinder-derived correlation state.

It must not rewrite authoritative PCAP or source FW journal history.

Pathfinder credentials/trust, if exposed to a Hunter component, must be narrowly scoped and must not grant ZFS host, FW management, retention/destruction, or unrelated Stronghold authority.

## External UI Boundary

The External UI Jail remains read-only with respect to authoritative Stronghold history and authoritative processing state.

The UI may search, filter, correlate, present timelines, pivot to packets, retrieve approved PCAP subsets, export, and create authorized notes/workspace artifacts.

It may display Pathfinder enrichment with explicit provenance. It must not convert an analyst annotation or Pathfinder classification into rewritten Stronghold source history.

## Firewall Decision Correlation

A first-class Net-Hunter capability should correlate:

```text
PCAP observation
    +
Traffic Decision Journal
    +
configuration generation
    +
Access / endpoint context where available
    +
Pathfinder intelligence where available
```

so the operator can answer what packet/session was observed, which rule matched, what exact rule content existed, what authorization occurred, route/WAN/NAT behavior, final disposition, NOT_PERFORMED work, endpoint/Access context, and what intelligence was later or contemporaneously associated.

Historical queries use the configuration generation that was active for the decision, not today's configuration with the same Policy ID.

## Timeline Views Are Derived

Net-Hunter may create correlated timelines across journals, configuration operations, HA transitions, path changes, Access/Agent state, Pathfinder intelligence events, and packet/flow activity.

```text
timeline presentation
!=
authoritative journal
```

A timeline entry preserves references to its authoritative and external sources.

## Search Result State and Confidence

Where relevant, query results should expose source/processing confidence and completeness such as:

```text
PCAP verification state
source-journal verification state
derived decoder / generation
index completeness
clock confidence
provenance type
Pathfinder Record ID / interpretation generation
Pathfinder data freshness
```

Stronghold must not flatten degraded, partial, indirect, external, stale, or unknown states into a generic successful result.

## Backlog Dimensions

Net-Hunter should distinguish separate backlog categories:

```text
TRANSFER BACKLOG
INGEST BACKLOG
PROCESSING BACKLOG
REPROCESSING BACKLOG
INDEX REBUILD BACKLOG
PATHFINDER ENRICHMENT / SYNC BACKLOG
```

A generic undifferentiated backlog number is insufficient.

## Resource Priorities

Net-Hunter is not inline, but current history preservation still outranks historical reinterpretation and enrichment.

Conceptual priority direction:

```text
1. receive current history
2. independently verify / durably commit
3. essential source catalog / source journal processing
4. current/recent derived processing and indexing
5. interactive hunt/query
6. historical reprocessing
7. Pathfinder / other optional enrichment
```

Historical reprocessing or intelligence enrichment must not starve current ingest or undermine authoritative preservation.

## PCAP Pivot Invariant

> **Where authoritative packet history still exists, a traffic-derived Hunter result should retain enough lineage to locate the packet segment or segments from which it was derived.**

This includes Pathfinder-enriched results.

```text
Pathfinder / flow / protocol / decision result
        ↓
source references
        ↓
PCAP segment(s)
        ↓
packet/time range
```

## Export Provenance

A PCAP export produced from a hunt is a derived extract, even when it contains exact source bytes.

Export metadata should preserve, as applicable, source segment IDs, source Appliance ID, selected time/filters, export generation time, exporting user/authority, integrity information, and Pathfinder Record references if intelligence was part of the selection criteria.

```text
exported PCAP
!=
original authoritative segment
```

## Net-Hunter Search Completeness Invariant

The operational answer to "nothing found" must reflect what Stronghold can actually establish.

Stronghold must distinguish:

```text
NO_MATCH_IN_COMPLETE_INDEX
INDEX_INCOMPLETE
PROCESSING_PENDING
DECODER_NOT_AVAILABLE
PATHFINDER_ENRICHMENT_PENDING
PATHFINDER_UNAVAILABLE_OR_STALE
SOURCE_HISTORY_UNAVAILABLE
SOURCE_HISTORY_DESTROYED
CAPTURE_GAP
```

Exact names remain future contract work, but the semantic distinction is mandatory.

## Hard Invariants

> **Authoritative PCAP and source journals remain the Stronghold source of truth. Derived records and indexes exist to locate, correlate, explain, enrich, and efficiently interrogate that history.**

> **Derived databases/indexes are rebuildable and must never become the only copy of a historical fact that Stronghold claims came from packet observation.**

> **Every material derived traffic fact retains provenance sufficient to identify the source appliance, source history object, processing generation/decoder, relevant authoritative packet/journal lineage, and any external intelligence record used.**

> **Reprocessing creates new or superseding derived interpretation without changing the original observation or falsely representing later understanding as knowledge Stronghold possessed at the original event time.**

> **Search coverage is explicit. An incomplete or rebuilding index or intelligence-enrichment state must not produce an unqualified `no results` answer.**

> **Where authoritative packet content remains retained, traffic-derived results should support a pivot back toward the applicable PCAP source.**

## Truth Separations

```text
database record                 != authoritative packet
no indexed result               != no traffic
index unavailable               != history unavailable
reprocessing result             != original knowledge
timeline                        != authoritative journal
exported PCAP                   != original PCAP segment
current hostname association    != historical hostname association
direct observation              != correlated association
processing failed               != source history lost
source history available        != index complete
Pathfinder record exists        != observable malicious
Pathfinder match                != IDS detection
Pathfinder enrichment           != original Stronghold knowledge
retrospective match             != historical real-time detection
Pathfinder unavailable          != observable trusted
```

## Scope

This architecture is intentionally later than Phase 0.

Phase 0 remains the Traffic Observation Foundation and does not implement the complete Net-Hunter record/index/search/reprocessing or Pathfinder-enrichment system merely because this contract exists.

The eventual database/index technology, physical schemas, partitioning strategy, exact Flow ID contract, decoder framework, query language, Pathfinder correlation schema, and UI/API representation remain future implementation decisions subject to measurement and explicit approval.
