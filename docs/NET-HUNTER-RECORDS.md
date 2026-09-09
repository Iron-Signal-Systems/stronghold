# Stronghold Net-Hunter Records, Search, and Reprocessing Architecture

## Purpose

This document defines the current architectural direction for Stronghold Net-Hunter records, indexing, search, correlation, query coverage, and historical reprocessing.

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

        ↓ derive

SEARCHABLE / DERIVED

segments
flows / sessions
protocol records
address / name relationships
decision correlations
indexes
timelines
search facets
```

If derived state is lost, Stronghold rebuilds it from authoritative source history where that source remains available.

If authoritative PCAP or source journals are lost, Stronghold reports actual historical loss. Rebuilding an index must never be presented as recovering source history that no longer exists.

## Authority Boundaries

Stronghold Net-Hunter maintains three distinct classes of information:

```text
WHAT WAS PRESENTED
    authoritative packet history / PCAPNG

WHAT STRONGHOLD DID
    authoritative FW and Hunter journals

WHAT STRONGHOLD UNDERSTOOD
    derived records, correlation, enrichment, and indexes
```

Derived databases and search indexes are rebuildable products of authoritative history.

They must never silently become the only copy of a fact that Stronghold claims originated from packet observation.

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

The exact schema remains future work.

The segment catalog must remain sufficient to narrow candidate PCAP history even when higher-level derived indexes are unavailable or incomplete.

### Traffic / Record Catalog

The traffic catalog contains derived records used to search, correlate, explain, and pivot into authoritative history.

It should not require a database row for every captured packet merely to make traffic searchable.

The preferred architecture is:

```text
PCAP
    authoritative packet detail

FLOW / SESSION RECORDS
    efficient traffic search

PROTOCOL / EVENT RECORDS
    higher-level derived facts

PCAP LOCATORS
    references back toward authoritative packet history
```

## Row-Per-Packet Is Not the Primary Search Model

Stronghold does not make a row-per-packet database the primary search architecture.

At meaningful packet rates, a required database row for every Ethernet frame creates unnecessary storage, indexing, and write amplification and risks turning the derived database into the practical source of truth.

Packet-level detail remains in authoritative PCAPNG.

Derived structures exist to narrow the search space and explain activity.

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
capture segment references
```

Not every field applies to every flow.

Stronghold must not force non-IP traffic into an IP-flow schema merely for implementation convenience.

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

Example conceptual LLDP record:

```text
Origin Appliance ID
source interface
observed chassis ID
observed port ID
system name
first seen
last seen
PCAP references
decoder/version provenance
```

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
```

For example, one IP may legitimately correlate to different MAC addresses during different intervals.

Stronghold must preserve those intervals instead of overwriting a historical relationship with the newest value.

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
```

### Direct and Correlated Facts

Stronghold distinguishes directly observed protocol facts from later correlation or external/user context.

Conceptual provenance classes include:

```text
DIRECT
    parsed directly from source packet or authoritative journal

CORRELATED
    derived by combining direct observations

EXTERNAL
    supplied by a later approved enrichment source

USER_SUPPLIED
    analyst annotation/context

NOT_KNOWN
```

Exact names remain future schema work.

A DNS-associated hostname must not be represented as directly observed TLS SNI merely because both refer to the same IP.

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
```

Search results should support pivots such as:

```text
IP
 ↓
flows
 ↓
policy decision
 ↓
NAT / WAN / route
 ↓
name/address relationships
 ↓
PCAP
```

CLI/UI/API design should hide database mechanics without hiding the underlying source and provenance.

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
```

Actual indexing strategy is benchmark-driven and remains implementation work.

## Time-Range Pruning

Most investigations have a bounded time range.

Net-Hunter should use segment metadata and later partition/index structures to narrow historical search early rather than scanning the full retained history when unnecessary.

The exact time-partitioning/sharding strategy remains implementation work.

## Database / Index Technology Is Not Yet Selected

Stronghold does not currently bind Net-Hunter to PostgreSQL, OpenSearch, ClickHouse, SQLite, a custom index, or another specific database technology.

Requirements are frozen before technology selection.

Any candidate implementation must be evaluated against:

```text
authority boundaries
record types
write / ingest rate
query patterns
retention behavior
rebuild behavior
expected scale
query latency
failure behavior
operational complexity
```

The architecture must not be reshaped merely to fit a convenient database product.

## Indexes Are Rebuildable

> **An index may be deleted and rebuilt without altering authoritative source history.**

Example:

```text
index corrupt
    ↓
mark index unavailable / degraded
    ↓
rebuild derived index
    ↓
source PCAP and source journals remain unchanged
```

Stronghold should distinguish:

```text
Search index: REBUILDING
Authoritative PCAP: AVAILABLE
```

from actual source-history unavailability.

## Query Coverage Is Explicit

> **Stronghold must never present an incomplete index as complete history.**

If authoritative packet history is retained through 18:00 but an index is complete only through 16:42, a query covering 00:00–18:00 must expose the incomplete coverage.

Conceptually:

```text
Requested:
    00:00–18:00

Indexed:
    00:00–16:42

Unprocessed / not indexed:
    16:42–18:00
```

A zero-result answer from an incomplete index must not be represented as proof that no matching traffic exists.

Net-Hunter must distinguish at least conceptually:

```text
not observed
not captured
captured but not processed
processed but not decoded
decoded but not indexed
indexed and no result
history destroyed
history unavailable
```

These states answer different operational questions and must not be collapsed.

## Search Fallback to Authoritative Catalogs

When derived indexes are unavailable or incomplete, Net-Hunter should retain a slower path through the segment catalog and authoritative source metadata.

Conceptually:

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

Where available from observation metadata, Hunter should preserve searchable facts such as:

```text
EtherType
IP protocol number
MAC addresses
interface
VLAN
time
frame / packet size
```

along with explicit decoder state.

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

Authoritative PCAP/source journals allow Net-Hunter to reinterpret old history with newer decoders/correlators.

Reprocessing never rewrites the original observation.

Example:

```text
Observation time:
    2028-06-14

Original processing:
    DECODER_NOT_AVAILABLE

Reprocessed:
    2030-01-09

Decoder:
    v4

New derived fact:
    ...
```

Stronghold must not present a later derived fact as knowledge Stronghold possessed at the original observation time.

### Processing and Decoder Versioning

Material derived records should preserve, as applicable:

```text
source object identity
derived-record generation
decoder / correlator version
processing time
supersession relationship
```

A later decoder may supersede an earlier interpretation without changing the source object.

### Targeted Reprocessing

Net-Hunter should eventually support targeted reprocessing by bounded scope such as:

```text
time range
Appliance ID
VLAN
segment range
protocol / decoder family
```

Targeted reprocessing operations belong in the Hunter Processing Journal.

### Full Reprocessing and Derived Generations

For significant decoder/schema changes, Stronghold should prefer building a new derived generation rather than leaving partially migrated current state.

Conceptually:

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

## Processing Failures Are Journaled

If processing fails after authoritative ingest succeeds, the authoritative segment remains committed.

The Hunter Processing Journal records the processing outcome and retry history.

Examples:

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
```

Later success does not erase the earlier failure.

## Record Processing Jail Boundary

The Record Processing Jail may:

```text
READ authoritative PCAP
READ authoritative/source journals
WRITE derived records
WRITE indexes
WRITE processing metadata
```

It must not rewrite authoritative PCAP or source FW journal history.

If normalized or intermediate representations are useful, they are derived artifacts with explicit lineage.

## External UI Boundary

The External UI Jail remains read-only with respect to authoritative Stronghold history and authoritative processing state.

The UI may:

```text
search
filter
correlate
present timelines
pivot to packets
retrieve approved PCAP subsets
export
create authorized notes / workspace artifacts
```

It must not rewrite authoritative PCAP, source journals, or derived records merely to make an event appear corrected.

User annotations remain separate user-supplied context.

## Firewall Decision Correlation

A first-class Net-Hunter capability should correlate:

```text
PCAP observation
    +
Traffic Decision Journal
    +
configuration generation
```

so the operator can answer:

```text
What packet/session was observed?
Which rule matched?
What exact rule content existed at that generation?
What authorization result occurred?
Was routing performed?
Which route was selected?
Which WAN was selected?
What NAT occurred?
What was the final disposition?
What processing was NOT_PERFORMED?
```

Historical queries use the configuration generation that was active for the decision, not merely today's configuration with the same Policy ID.

## Historical Configuration Correlation

The isolated FW Configuration Backup Jail preserves committed configuration generations for recovery and historical reconstruction.

Net-Hunter should be able to correlate a decision referencing generation `N` with policy/routes/NAT/WAN objects as they actually existed in generation `N`.

```text
Policy ID stable identity
!=
current policy content
```

## Timeline Views Are Derived

Net-Hunter may create correlated timelines across journals, configuration operations, HA transitions, path changes, and packet/flow activity.

A timeline is a presentation/index structure whose entries retain references to authoritative sources.

```text
timeline presentation
!=
authoritative journal
```

## Search Result State and Confidence

Where relevant, query results should expose source/processing confidence and completeness such as:

```text
PCAP verification state
source-journal verification state
derived decoder / generation
index completeness
clock confidence
provenance type
```

Stronghold must not flatten degraded, partial, indirect, or unknown states into a generic successful result.

## Backlog Dimensions

Net-Hunter should distinguish separate backlog categories:

```text
TRANSFER BACKLOG
    history still waiting on FW

INGEST BACKLOG
    received but not durably committed/verified

PROCESSING BACKLOG
    committed source history not yet decoded/indexed

REPROCESSING BACKLOG
    scheduled historical reinterpretation

INDEX REBUILD BACKLOG
    derived index rebuild work
```

A generic undifferentiated backlog number is insufficient.

## Resource Priorities

Net-Hunter is not inline, but current history preservation still outranks historical reinterpretation.

Conceptual priority direction:

```text
1. receive current history
2. independently verify / durably commit
3. essential source catalog / source journal processing
4. current/recent derived processing and indexing
5. interactive hunt/query
6. historical reprocessing
7. deeper optional enrichment
```

Exact scheduling remains implementation work.

Historical reprocessing must not starve current ingest or undermine authoritative preservation.

## PCAP Pivot Invariant

> **Where authoritative packet history still exists, a traffic-derived Hunter result should retain enough lineage to locate the packet segment or segments from which it was derived.**

Exact packet offsets are not necessarily required in the first implementation, but the architecture must preserve sufficient source references to move from a derived result back toward authoritative PCAP.

Conceptually:

```text
Flow / protocol / decision result
        ↓
source references
        ↓
PCAP segment(s)
        ↓
packet/time range
```

## Export Provenance

A PCAP export produced from a hunt is a derived extract, even when it contains exact source bytes.

Export metadata should preserve as applicable:

```text
source segment IDs
source Appliance ID
selected time / filters
export generation time
exporting user / authority
integrity information
```

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
SOURCE_HISTORY_UNAVAILABLE
SOURCE_HISTORY_DESTROYED
CAPTURE_GAP
```

Exact names remain future contract work, but the semantic distinction is mandatory.

This protects the product from the failure mode where an operator is told that something did not happen merely because one software version, parser, index, or processing stage did not record it.

## Hard Invariants

> **Authoritative PCAP and source journals remain the source of truth. Derived records and indexes exist to locate, correlate, explain, and efficiently interrogate that history.**

> **Derived databases/indexes are rebuildable and must never become the only copy of a historical fact that Stronghold claims came from packet observation.**

> **Every material derived traffic fact retains provenance sufficient to identify the source appliance, source history object, processing generation/decoder, and relevant authoritative packet/journal lineage.**

> **Reprocessing creates new or superseding derived interpretation without changing the original observation or falsely representing later understanding as knowledge Stronghold possessed at the original event time.**

> **Search coverage is explicit. An incomplete or rebuilding index must not produce an unqualified `no results` answer.**

> **Where authoritative packet content remains retained, traffic-derived results should support a pivot back toward the applicable PCAP source.**

## Truth Separations

Stronghold preserves these distinctions:

```text
database record
!=
authoritative packet

no indexed result
!=
no traffic

index unavailable
!=
history unavailable

reprocessing result
!=
original knowledge

timeline
!=
authoritative journal

exported PCAP
!=
original PCAP segment

current hostname association
!=
historical hostname association

direct observation
!=
correlated association

processing failed
!=
source history lost

source history available
!=
index complete
```

## Scope

This architecture is intentionally later than Phase 0.

Phase 0 remains the Traffic Observation Foundation and does not implement the complete Net-Hunter record/index/search/reprocessing system merely because this contract exists.

The eventual database/index technology, physical schemas, partitioning strategy, exact Flow ID contract, decoder framework, query language, and UI/API representation remain future implementation decisions subject to measurement and explicit approval.
