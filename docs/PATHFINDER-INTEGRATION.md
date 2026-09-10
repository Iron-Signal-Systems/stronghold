# Stronghold Pathfinder Integration Architecture

## Purpose

This document defines the architectural relationship between the **Stronghold platform** and **Iron Signal Systems Pathfinder**.

Stronghold and Pathfinder are separate ISS systems with separate source truth and authority. They may be tightly integrated, but neither silently becomes the other's database, policy engine, or historical authority.

The governing relationship is:

> **Pathfinder owns the organization's record and interpretation of threat intelligence. Stronghold owns observation, access/enforcement decisions, operational history, and the actions performed in the Stronghold environment.**

Pathfinder is an optional first-class ISS intelligence peer for Stronghold. Stronghold must remain able to perform its core capture, firewalling, local enforcement, and ordinary access-control responsibilities when Pathfinder is unavailable.

## Platform Relationship

Conceptually:

```text
                         PATHFINDER
                  threat-intelligence authority
                 record / interpretation / context
                            |
                 authenticated/versioned exchange
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
   STRONGHOLD FW      STRONGHOLD ACCESS   NET-HUNTER
          |                 |                  |
          |                 |                  +-- historical correlation
          |                 |                  +-- retrospective matching
          |                 |                  +-- reprocessing
          |                 |
          |                 +-- risk/trust context input
          |
          +-- traffic / observable enrichment
          +-- operator context
          +-- IDS/IPS contextual input
```

Stronghold Agent does not normally communicate directly with Pathfinder. Endpoint policy and access decisions remain distributed through Stronghold Access and the approved Stronghold control contracts.

## Authority Boundary

Pathfinder provides intelligence and interpretation. Stronghold determines what that intelligence means for the customer's Stronghold policy and whether any action is authorized.

Mandatory separations:

```text
Pathfinder observable match        != Stronghold enforcement action
Pathfinder malicious classification != compromise proven
Pathfinder confidence HIGH          != automatic DROP
Pathfinder risk signal              != automatic Access REVOKE
IDS finding                         != Pathfinder intelligence confirmed
Stronghold observation              != Pathfinder malicious classification
Stronghold observable submitted     != Pathfinder record accepted
new Pathfinder interpretation       != knowledge Stronghold had at observation time
```

A Pathfinder record may be a high-value policy input. It is never an unqualified bypass around Stronghold policy, change authority, access-control rules, or journal attribution.

## Pathfinder Intelligence Inputs

Subject to the Pathfinder contract and customer's configured sources, Stronghold may consume Pathfinder intelligence associated with observables such as:

```text
IP address
network / prefix
domain / FQDN
URL or URI where legitimately available
certificate fingerprint
file hash where legitimately available
hostname or infrastructure identity where appropriately sourced
protocol/application artifact
malware / campaign / actor association
classification
confidence
source provenance
first-seen / last-seen context
Pathfinder Record ID
Pathfinder interpretation generation/version
```

Exact fields and vocabulary remain owned by Pathfinder and the future versioned Stronghold↔Pathfinder contract.

Stronghold must preserve the Pathfinder source identity and interpretation version used for any displayed enrichment or policy decision.

## Stronghold FW Enrichment

Stronghold FW may enrich recent operational traffic and decision views with Pathfinder context without converting that enrichment into physical observation truth.

Conceptually:

```text
Stronghold observed:
    source / destination
    VLAN / zone
    policy
    route
    WAN
    NAT
    final result
    packet history reference

Pathfinder adds:
    classification
    confidence
    campaign / malware association
    source / provenance
    Pathfinder Record ID
```

A useful operator view may therefore show:

```text
SOURCE
    FIN-PC-17
    VLAN 120

DESTINATION
    185.x.x.x
    TCP/443

STRONGHOLD
    Policy P-183
    WAN1
    FORWARDED

PATHFINDER
    Classification: known C2 infrastructure
    Confidence: HIGH
    Record: PF-...
```

The display must continue to distinguish Stronghold facts from Pathfinder interpretation.

### Inline Policy Use

Pathfinder intelligence may later be usable as an explicit Stronghold policy fact, subject to separately frozen semantics.

Examples might include:

```text
Pathfinder classification
confidence threshold
record/source class
age/freshness
campaign association
customer-approved intelligence set
```

However:

```text
Pathfinder says malicious
!=
Stronghold automatically blocks
```

Any automatic enforcement based on Pathfinder must be deliberately configured, attributable, bounded, and journaled as a Stronghold action under a known policy generation.

## Net-Hunter Historical Enrichment and Retrospective Matching

Net-Hunter is a particularly important Pathfinder integration point because new intelligence can be applied to previously preserved Stronghold history.

Conceptually:

```text
new Pathfinder interpretation
        |
        v
observable newly classified / associated
        |
        v
Net-Hunter historical search/reprocessing
        |
        +-- prior flows
        +-- prior endpoints
        +-- prior sessions
        +-- relevant packet segments
        +-- related Stronghold decisions
```

Example:

```text
Pathfinder Record PF-98471
    observable newly associated with C2

Net-Hunter retrospective result:
    first Stronghold observation: T1
    affected endpoints: ...
    sessions: ...
    source PCAP: available
```

This must preserve the historical knowledge boundary:

```text
observed then
!=
interpreted as malicious then

historically derivable now
!=
known then
```

A new Pathfinder interpretation creates new derived Stronghold/Hunter correlation. It does not rewrite the original PCAP, FW journal, old IDS finding, or prior decision as though the intelligence had existed earlier.

## IDS / IPS Complement

Pathfinder is intended to **complement** Stronghold IDS/IPS rather than replace the IDS engine or become the IDS ruleset.

Conceptually:

```text
                 STRONGHOLD OBSERVATION
                          |
                          v
                     IDS ANALYSIS
                          |
              +-----------+-----------+
              |                       |
              v                       v
       protocol/content           PATHFINDER
          detection              intelligence
              |                       |
              +-----------+-----------+
                          |
                          v
                   STRONGHOLD CORRELATION
```

A Stronghold/Hunter detection may become more useful when combined with Pathfinder context such as known infrastructure, certificate/hash reputation, campaign association, confidence, recency, or related observables.

Example:

```text
IDS
    suspicious TLS/application behavior

Pathfinder
    destination associated with known C2
    certificate fingerprint previously reported
    domain related to a current campaign

Stronghold
    source is a finance endpoint
    first contact from this device
    policy allowed the flow
    packet history preserved
```

Pathfinder context may later contribute to authorized IPS behavior for new/future flows, but:

```text
IDS detection + Pathfinder match
!=
automatic enforcement authority
```

Stronghold policy still determines whether an IPS action is permitted and records the actual action performed.

## Stronghold Access Enrichment

Pathfinder may provide a risk/trust input to Stronghold Access policy evaluation.

For example, a Stronghold-observed endpoint/session relationship combined with high-confidence Pathfinder intelligence may cause Stronghold Access to reevaluate a current Access Session.

Potential configured outcomes may later include:

```text
require MFA reauthentication
reduce granted resource scope
shorten lease duration
revoke privileged access
REVOKE an Access Session
request network-admission reevaluation / CoA
place endpoint into a restricted/quarantine policy
```

The exact policy model remains later work.

Mandatory boundary:

```text
Pathfinder risk signal
!=
automatic Stronghold Access revocation
```

Stronghold Access remains the authorization authority and must record why it granted, denied, restricted, or revoked access.

## Stronghold Agent Boundary

Stronghold Agent should normally receive the result of Stronghold Access policy evaluation rather than query Pathfinder directly.

Preferred relationship:

```text
Pathfinder
    -> Stronghold Access
        -> signed/versioned Agent policy
            -> Stronghold Agent
```

This avoids creating a Pathfinder trust relationship, credential set, and intelligence-query dependency on every managed endpoint.

If a future use case requires direct Agent↔Pathfinder communication, it requires a separately approved contract and threat model.

## Stronghold-to-Pathfinder Feedback

Stronghold may discover observables and relationships that are useful to Pathfinder.

Potential submissions may include, according to explicit configuration and data-governance rules:

```text
observed IP / domain
certificate fingerprint
file hash where legitimately available
protocol/application artifact
IDS finding reference
historical relationship
Stronghold source identifier / provenance
observation time
```

Stronghold submission is an observation/input, not a malicious classification.

```text
Stronghold observed observable
!=
observable malicious
```

Pathfinder independently records, correlates, interprets, and classifies according to Pathfinder's own authority model.

## Provenance and Lineage

Every material Pathfinder-derived Stronghold fact should preserve enough lineage to establish:

```text
Pathfinder Record ID
Pathfinder source/tenant/system identity as applicable
interpretation/classification generation or version
retrieval/receipt time
source observable
Stronghold object/session/flow being enriched
Stronghold policy generation if used for enforcement
```

Stronghold must not flatten Pathfinder intelligence into an unattributed boolean such as `malicious=true` when richer provenance is required to explain a decision later.

## Journaling

When Pathfinder intelligence materially influences a Stronghold action, Stronghold journals should preserve enough context to explain that fact.

Potential examples include:

```text
Traffic Decision Journal
    Pathfinder input referenced by policy decision
    actual FW/IPS result

Trust / Identity Journal
    Access reevaluation/revocation influenced by Pathfinder context

Administrative Journal
    operator configured Pathfinder-driven policy or changed integration/trust

System / Health Journal
    Pathfinder integration availability/freshness/backlog/trust failure

Hunter Processing Journal
    retrospective Pathfinder enrichment/reprocessing generation
```

Pathfinder's own records remain Pathfinder authority. Stronghold journals record what Stronghold consumed and did.

## Availability and Failure Behavior

Pathfinder is not a mandatory runtime dependency for basic Stronghold operation.

A Pathfinder outage must not silently stop:

```text
AF_XDP packet acquisition
PCAP durability
ordinary FW forwarding/enforcement
local FW journal commit
Net-Hunter authoritative history ingest
existing Agent WFP enforcement under valid policy
ordinary Access operation that does not explicitly require fresh Pathfinder input
```

Potential integration states may include:

```text
AVAILABLE
DEGRADED
STALE
UNAVAILABLE
AUTHORIZATION_FAILED
TRUST_FAILED
SYNC_BACKLOG
```

Exact vocabulary remains later implementation work.

If a customer deliberately configures a policy that requires fresh Pathfinder intelligence, the fail behavior must be explicit, scoped, and journaled. Stronghold must not silently convert `Pathfinder unavailable` into `Pathfinder says safe`.

```text
Pathfinder unavailable
!=
observable trusted
```

## Resource Priority

Pathfinder enrichment, retrospective matching, and intelligence synchronization are secondary to Stronghold live capture and essential enforcement responsibilities.

On Stronghold FW, intelligence lookup must not become an unbounded synchronous per-packet dependency that can starve AF_XDP acquisition or authoritative PCAP writes.

Preferred patterns include bounded local intelligence state, cached/versioned enrichment, asynchronous correlation, or explicitly qualified policy lookups where needed.

Net-Hunter may perform deeper retrospective Pathfinder matching because it is not the live forwarding path, while still protecting authoritative ingest and durability.

## Security and Trust

Stronghold↔Pathfinder integration requires an explicit authenticated trust relationship.

Future contract requirements include:

```text
mTLS or approved authenticated transport
explicit peer/system authorization
purpose-specific credentials
versioned API/schema contract
replay/downgrade protection where applicable
least-privilege scopes
credential rotation/revocation
source/tenant identity binding
health/freshness reporting
```

Pathfinder integration credentials are purpose-separated from Stronghold HISTORY, HA, journal-signing, management, IDS/inspection, WireGuard, and site-tunnel credentials.

A valid Pathfinder transport credential does not grant Stronghold configuration or history-destruction authority.

## Truth Boundaries

Never collapse these distinctions:

```text
Pathfinder record exists              != observable malicious
Pathfinder classification malicious   != compromise proven
Pathfinder confidence HIGH            != Stronghold action authorized
Pathfinder match                       != IDS detection
IDS finding                            != Pathfinder confirmation
Stronghold observation                 != Pathfinder intelligence
Stronghold submission accepted        != malicious classification
Pathfinder unavailable                != observable trusted
Pathfinder data stale                 != current interpretation known
new Pathfinder interpretation         != original historical knowledge
retrospective match                    != historical real-time detection
intelligence synchronized             != all Stronghold policy updated
Access reevaluation requested         != Access Session revoked
IPS action requested                  != IPS action performed
```

## Qualification Requirements

Before production Pathfinder integration is promoted into an implementation phase, freeze and validate at minimum:

```text
Stronghold↔Pathfinder trust/enrollment
API/protocol/schema versioning
observable identifiers and normalization
Pathfinder Record ID/reference semantics
classification/confidence/freshness representation
provenance/lineage contract
FW enrichment caching/update model
whether any policy lookups are synchronous
Access risk-input semantics
IDS correlation semantics
Net-Hunter retrospective matching/reprocessing
Stronghold-to-Pathfinder submission policy
privacy/data-governance boundaries
rate limiting/backpressure
failure/holdover/stale-data behavior
HA behavior where applicable
journaling and audit fields
credential purpose/rotation/revocation
performance/scaling
upgrade/rollback compatibility
```

## Scope

This document defines architecture only.

Pathfinder integration is not part of Stronghold Phase 0. Phase 0 remains focused on proving the AF_XDP physical observation pipeline, durable PCAP, storage behavior, truthful loss accounting, and verified Net-Hunter history transfer.

No Pathfinder integration code, synchronous intelligence dependency, automatic blocking policy, automatic Access revocation, or IDS/IPS implementation is pulled into Phase 0 by this document.
