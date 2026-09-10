# Stronghold IDS / TLS Inspection Architecture

## Purpose

This document defines the current future architecture for Stronghold TLS/application inspection and IDS/IPS integration.

**IDS/IPS implementation remains deferred.** This document establishes the architectural boundaries that any later implementation must preserve. It does not select an IDS engine, ruleset format, final proxy implementation, or complete IPS enforcement contract.

The governing principle is:

> **Useful inspection of encrypted application traffic requires plaintext somewhere, but decrypted traffic must not become a second uncontrolled historical datastore or a mandatory dependency for ordinary Stronghold forwarding.**

Stronghold therefore separates authoritative wire observation, selective proxy decryption, transient IDS transport, persisted detection findings, and external threat-intelligence enrichment.

## Authority and Data Classes

Stronghold maintains distinct classes of information:

```text
AUTHORITATIVE WIRE HISTORY
    original physical-interface PCAPNG
    encrypted TLS/QUIC/application bytes as actually observed

EPHEMERAL DECRYPTED INSPECTION DATA
    plaintext created only by an explicitly selected proxy/inspection path
    non-authoritative
    volatile on FW
    volatile in the IDS processing path except where policy explicitly permits persistence

PERSISTENT IDS FINDINGS
    derived detection/correlation results
    encrypted at rest
    linked back to applicable flow/source history

OPTIONAL RETAINED INSPECTION CONTEXT
    explicit policy only
    encrypted at rest
    separately retained/authorized

PATHFINDER INTELLIGENCE
    external ISS threat-intelligence record / interpretation
    separate authority from Stronghold observation and IDS findings
    referenced by record/version/provenance when used
```

The following distinctions are mandatory:

```text
original encrypted PCAP        != decrypted inspection history
IDS finding                    != authoritative packet
IDS finding retained           != full decrypted session retained
inspection feed lost           != packet observation lost
decrypted in FW memory         != decrypted retained on FW
Pathfinder match               != IDS detection
IDS finding                    != Pathfinder confirmation
Pathfinder classification      != Stronghold enforcement action
```

## Selective Proxy Architecture

TLS/application inspection is an explicitly selected policy behavior, not a prerequisite for normal forwarding.

Conceptual outbound path:

```text
CLIENT
   │
   │ TLS SESSION A
   ▼
┌────────────────────────────────────┐
│           STRONGHOLD FW            │
│                                    │
│ authoritative wire capture         │
│        │                           │
│ selective TLS/application proxy    │
│        │                           │
│ plaintext only in volatile memory  │
│        ├───────────────┐           │
│        │               │           │
│        ▼               ▼           │
│ TLS SESSION B     INSPECTION FEED  │
└────────┬───────────────┬───────────┘
         │               │ encrypted + authenticated
         ▼               ▼
      SERVER       NET-HUNTER IDS JAIL
```

An inspected connection remains encrypted on both sides of Stronghold. Stronghold does not require HTTPS downgrade to perform enterprise proxy inspection.

```text
TLS proxied
!=
TLS downgraded
```

## HSTS and Application Compatibility

HSTS does not itself prohibit an enterprise proxy when the managed client explicitly trusts the configured inspection CA and the connection remains HTTPS.

More difficult cases include:

```text
certificate pinning
mutual TLS
applications with independent trust stores
QUIC / HTTP/3 behavior
protocols that do not support the selected proxy model
future encrypted-handshake/application mechanisms
```

Stronghold must not fake successful inspection when an application cannot be safely proxied.

Inspection policy must be able to represent outcomes such as:

```text
PROXY_INSPECTED
BYPASSED
NOT_AVAILABLE
UNSUPPORTED
FAILED
```

with explicit reasons.

Inspection bypass never means the original traffic was not observed by the Stronghold physical-interface capture path.

## Inspection CA and Trust Separation

The inspection certificate authority is purpose-specific.

Stronghold should favor a customer-controlled inspection trust architecture rather than any universal Iron Signal Systems interception root.

Inspection-signing credentials remain separate from:

```text
Stronghold appliance identity
FW↔Hunter HISTORY transport identity
HA identity
management/API identity
journal-signing identity
IDS transport identity
Pathfinder integration identity
```

Private inspection CA material should be hardware protected/non-exportable where supported, subject to later recovery and key-management design.

## Origin Certificate Validation

Stronghold independently validates the real origin-server certificate on the server-facing TLS connection.

A valid Stronghold-generated client-side certificate must never hide an invalid origin certificate.

```text
client-side generated certificate valid
!=
origin certificate valid
```

Origin failures such as hostname mismatch, expiration, untrusted chain, revocation, or other certificate errors are handled according to explicit policy and reported truthfully.

## Inbound Reverse-Proxy Direction

Inbound application inspection is a distinct proxy case.

```text
INTERNET CLIENT
      │ TLS
      ▼
STRONGHOLD
      │ TLS
      ▼
INTERNAL SERVER
```

The external and internal legs should normally remain encrypted. Stronghold does not create plaintext internal transport merely because it is inspecting application traffic.

The exact certificate-deployment and reverse-proxy model remains to be frozen for later implementation.

## FW Plaintext Retention Rule

> **Stronghold FW never intentionally persists decrypted inspection payload at rest.**

Decrypted payload on FW is limited to controlled volatile processing/transport buffers.

FW must not intentionally create:

```text
decrypted PCAP files
decrypted session spool files
plaintext retry queues
plaintext temporary files
plaintext retained debug artifacts
plaintext crash/core artifacts
```

where those artifacts contain inspected payload.

Original encrypted wire traffic remains eligible for authoritative PCAP retention because it is what the physical interface actually observed.

## Volatile Memory Handling

Plaintext exists transiently on the FW proxy and, after encrypted transport, inside the Hunter IDS processing boundary.

Implementation must account for accidental persistence through:

```text
swap
core dumps
panic/crash dumps
debug logging
support bundles
temporary files
memory-backed filesystems with persistence/swap behavior
```

The later implementation should use bounded process buffers and platform-appropriate memory/dump controls. Stronghold support/diagnostic tooling must treat inspection memory and crash artifacts as high-sensitivity material.

## Dedicated IDS / Inspection Transport

The transient decrypted inspection feed is sent from FW to Net-Hunter over a dedicated encrypted and mutually authenticated logical transport.

```text
FW inspection worker
       │
       │ mTLS / approved Stronghold crypto profile
       │ purpose-specific IDS/inspection identity
       ▼
Hunter IDS inspection endpoint
       │
       ▼
IDS / INSPECTION JAIL
```

There is no plaintext fallback.

Both sides require certificate validity, correct IDS/inspection credential purpose, and explicit peer authorization.

## IDS Transport Identity Separation

IDS/inspection transport credentials are separate from authoritative HISTORY transport credentials and Pathfinder integration credentials.

```text
HISTORY transport identity
!=
INSPECTION transport identity
!=
Pathfinder integration identity
```

## HISTORY vs INSPECTION Transport

Authoritative history and transient inspection are different data planes even if an initial hardware profile carries both over the same physical high-speed FW↔Hunter fabric.

```text
HISTORY
    authoritative
    durable
    verified
    retransmittable
    Hunter durable commit before ACK
    higher priority

INSPECTION
    non-authoritative decrypted copy
    transient
    separately authenticated/encrypted
    bounded/loss-aware
    lower priority than authoritative history
```

They require separate credentials, authorization, queues, resource accounting, health state, and failure state.

## Net-Hunter IDS / Inspection Jail

The future IDS function belongs in a distinct isolated jail rather than being silently folded into the existing Record Processing Jail.

```text
1. PCAP Data Ingest Jail
2. Record Processing Jail
3. External User Interface Jail
4. FW Configuration Backup Jail
5. IDS / Inspection Jail    [future / optional]
```

The IDS jail receives the transient inspection stream and performs deep detection/analysis.

It does not receive implicit authority over:

```text
FW configuration
Hunter host/root
ZFS pool administration
retention/hold administration
authoritative PCAP writes
FW configuration backups
key-management authority
Pathfinder configuration authority
```

The jail is intended to be replaceable/rebuildable without altering authoritative PCAP or source journals.

## Encrypted At-Rest IDS Domain

Anything derived from decrypted traffic and intentionally persisted by the IDS subsystem is sensitive at-rest data.

Net-Hunter therefore uses a dedicated encrypted inspection persistence domain.

Conceptually:

```text
Hunter ZFS
  stronghold/
    authoritative/
      pcap/
      journals/

    derived/
      records/
      indexes/
      intelligence-correlation/

    inspection/
      findings/
      retained-context/
```

The exact dataset names are illustrative, not frozen.

The FreeBSD host retains ZFS/key-management authority. The IDS jail receives only the mounted dataset access required for its role.

## Inspection Persistence Classes

Stronghold should preserve multiple explicit persistence classes rather than silently retaining full plaintext whenever IDS is enabled.

```text
FINDINGS_ONLY
    default direction
    persist detection metadata/result

BOUNDED_CONTEXT
    persist explicit bounded matched/context material
    encrypted at rest

FULL_DECRYPTED_SESSION
    future explicit high-sensitivity option only
    disabled by default
    separate authorization/retention/capacity implications
```

Exact names and availability remain to be frozen later.

## IDS Findings

A persistent IDS finding may include information such as:

```text
Finding ID
origin FW Appliance ID
Flow ID / session reference
inspection policy
inspection state
detector / engine identity
rule / signature identity
ruleset generation
classification
confidence
observation/detection time
source/destination context
original PCAP segment references
bounded matched/context material where explicitly permitted
processing generation
Pathfinder Record references where correlated later
```

A finding is derived interpretation and never replaces the authoritative encrypted-wire PCAP.

Because findings themselves may contain sensitive information such as URLs, hostnames, headers, filenames, usernames, payload fragments, or malware strings, the findings store is encrypted at rest even when full decrypted sessions are not retained.

## Pathfinder Intelligence Complement

Pathfinder is intended to complement Stronghold IDS/IPS, not replace the IDS detector, ruleset, or Stronghold policy engine.

Pathfinder remains a separate ISS threat-intelligence authority governed by `docs/PATHFINDER-INTEGRATION.md`.

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
                   CORRELATION / CONTEXT
```

Pathfinder may enrich an IDS finding or flow with context such as:

```text
observable classification
confidence
source provenance
first/last-seen context
campaign / malware association
related infrastructure
certificate/hash/domain relationships
Pathfinder Record ID
interpretation generation/version
```

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
    authoritative PCAP preserved
```

These remain separate facts.

```text
Pathfinder match
!=
IDS detection

IDS finding
!=
Pathfinder confirmation

IDS finding + Pathfinder match
!=
automatic IPS authority
```

Any later enforcement action based on the combination remains an explicit Stronghold policy decision and is journaled as what Stronghold actually did.

## Historical Reprocessing Limitation

If Stronghold does not retain decrypted sessions or TLS session secrets, later reprocessing of old encrypted PCAP cannot reconstruct the historical plaintext merely because the original wire PCAP still exists.

```text
encrypted historical PCAP
!=
historical plaintext available
```

Stronghold should not retain TLS session secrets by default merely to enable later retrospective decryption.

However, Pathfinder intelligence can still be applied retrospectively to historical flow/observable metadata and source packet references where the relevant observable is derivable from retained history.

```text
retrospective Pathfinder match
!=
historical plaintext reconstruction
```

## Inspection Retention

Inspection findings/context have retention independent from authoritative PCAP and other journal domains.

```text
AUTHORITATIVE PCAP
    retention policy A

IDS FINDINGS
    retention policy B

DECRYPTED RETAINED CONTEXT
    retention policy C

PATHFINDER-DERIVED CORRELATION
    derived/rebuildable lineage according to Hunter policy
```

```text
IDS finding expired
!=
authority to destroy authoritative PCAP
```

## IDS Resource Priority

Stronghold FW remains a firewall/capture appliance first.

On FW, the inspection system performs only the work needed to proxy the selected live connection and create the bounded transient inspection feed.

Deep IDS parsing/rule evaluation and richer Pathfinder correlation belong on Hunter where possible.

The inspection/intelligence path never outranks packet acquisition, active authoritative PCAP writes, essential segment finalization/integrity, essential forwarding/enforcement work, critical HA heartbeat/control, or authoritative history transfer needed to protect backlog.

Pathfinder lookups must not become an unbounded synchronous per-packet dependency on the FW dataplane.

## Inspection Backpressure and Failure

Normal IDS mode must not make Hunter processing latency a synchronous dependency for every production packet.

If the IDS jail or inspection transport cannot keep up, the default architecture is:

```text
production proxy session
    continues according to configured inspection policy

inspection feed
    degrades/drops in a bounded and measurable manner

Stronghold
    records INSPECTION_FEED_GAP / degraded coverage truthfully
```

A Pathfinder outage or stale intelligence condition is independently represented and does not masquerade as an IDS failure.

```text
Pathfinder unavailable
!=
IDS unavailable

IDS unavailable
!=
production forwarding unavailable
```

A future explicitly configured `MUST_INSPECT` or IPS policy may choose fail-closed behavior, but such behavior must be explicit, scoped, and separately qualified.

## IDS vs IPS

Deep Hunter analysis should not become a per-packet synchronous forwarding oracle.

The architecture intentionally avoids making every packet wait for Hunter deep IDS or Pathfinder analysis before forwarding.

Future IPS may use carefully bounded mechanisms such as:

```text
local FW proxy termination for specifically qualified immediate rules
Hunter detection producing an authorized dynamic enforcement fact for new/future flows
Pathfinder-enriched intelligence contributing to a policy input
explicit policy requiring inspection before continuation
```

The exact enforcement architecture remains deferred and must preserve Stronghold authorization, journaling, resource-priority, and failure-state rules.

```text
deep detection
!=
inline enforcement
```

## QUIC / HTTP/3

QUIC/HTTP/3 requires deliberate future handling.

Possible future policy behaviors may include native QUIC proxy/termination, explicit QUIC bypass, or policy-driven HTTP/3 suppression/fallback.

Stronghold must report which behavior occurred. It must not silently disable QUIC and imply native inspection support.

## Mutual TLS

Transparent interception of mutual TLS changes endpoint authentication semantics.

The default future architectural direction is therefore to bypass mTLS unless a specific service has an explicitly designed proxy/delegation model.

```text
mTLS observed
!=
safe to transparently intercept
```

## Certificate Pinning

Applications that intentionally pin a server certificate/public key should normally bypass or report unsupported inspection rather than Stronghold attempting generic pinning defeat.

## Inspection Coverage

Stronghold must not present `DPI ENABLED` as proof that encrypted traffic was actually inspected.

Inspection coverage should distinguish at least conceptually:

```text
traffic observed
traffic selected for inspection
traffic successfully proxied/decrypted
traffic bypassed by policy
traffic bypassed/unsupported due to application behavior
inspection feed delivered
inspection feed dropped/degraded
IDS processing completed
finding generated
Pathfinder enrichment available / unavailable / stale
Pathfinder correlation applied
```

An incomplete inspection feed, unsupported application, or unavailable intelligence source must remain visible as a coverage limitation.

## Support and Diagnostic Boundaries

Support bundles, debug logging, crash handling, and engineering access must preserve the support architecture in `docs/SUPPORT-DIAGNOSTICS.md`.

In particular:

- decrypted inspection payload is never silently included in ordinary support bundles;
- IDS memory/core dumps are high-sensitivity artifacts;
- debug logging must not casually dump plaintext payload;
- inspection at-rest keys are not exposed to support identities merely because they can diagnose the jail;
- Pathfinder integration credentials are not exposed through ordinary IDS diagnostics;
- temporary diagnostic actions must not create a plaintext spool.

## Health and Alerting

The observability architecture later needs independent IDS/inspection and Pathfinder-integration health such as:

```text
proxy availability
inspection certificate/trust health
inspection transport authorization
inspection feed throughput/drop/gap state
IDS jail health
processing backlog
ruleset/engine generation
inspection storage health
inspection coverage
Pathfinder integration availability
Pathfinder data freshness
Pathfinder sync/backlog state
```

Healthy forwarding/capture must not hide degraded inspection, and degraded inspection/intelligence must not be reported as packet-observation loss when the authoritative capture path remained healthy.

## Architecture Invariants

> **TLS/application inspection is an explicitly selected proxy function, not a prerequisite for ordinary Stronghold forwarding.**

> **An inspected connection remains encrypted on both sides of Stronghold. Plaintext exists only inside controlled volatile processing boundaries unless an explicit encrypted-retention policy permits selected IDS material to persist.**

> **Stronghold FW never intentionally persists decrypted inspection payload at rest.**

> **Stronghold FW transports decrypted inspection material to Net-Hunter only over a dedicated mutually authenticated encrypted inspection channel using purpose-specific IDS/inspection credentials and explicit peer authorization. There is no plaintext fallback.**

> **Net-Hunter IDS processing occurs in an isolated future IDS/Inspection Jail. Any inspection-derived material that persists is stored only in a dedicated encrypted-at-rest inspection domain whose key-management authority remains with the Hunter host.**

> **Normal IDS overload, Pathfinder unavailability, or Hunter unavailability does not silently backpressure ordinary production forwarding. Any fail-closed behavior must be explicit and scoped.**

> **Full decrypted sessions and TLS session secrets are not retained by default. Findings-only is the default architectural direction.**

> **Authoritative encrypted PCAP remains the wire-history authority. IDS findings and Pathfinder intelligence are separate forms of interpretation/context and retain provenance toward their sources.**

## Truth Boundaries

Never collapse these distinctions:

```text
HSTS enabled                       != enterprise TLS inspection impossible
TLS proxied                        != TLS downgraded
client inspection cert valid       != origin certificate valid
inspection bypassed                != traffic unobserved
inspection feed lost               != authoritative capture lost
decrypted in FW memory             != decrypted retained on FW
HISTORY transport identity         != INSPECTION transport identity
encrypted inspection transport     != authorized inspection peer
IDS jail access                    != ZFS key-management authority
IDS finding retained               != full decrypted session retained
original encrypted PCAP            != decrypted inspection history
encrypted historical PCAP          != historical plaintext available
IDS storage unavailable            != permission to spool plaintext
IDS unavailable                    != production forwarding unavailable
deep detection                     != inline enforcement
inspection configured              != inspection coverage complete
Pathfinder record exists           != observable malicious
Pathfinder match                   != IDS detection
IDS finding                        != Pathfinder confirmation
Pathfinder confidence HIGH         != IPS action authorized
retrospective intelligence match   != historical real-time detection
```

## Deliberately Unfrozen

The following remain future implementation decisions:

```text
IDS/IPS engine
ruleset/signature format
proxy implementation
exact TLS cryptographic profile
inspection CA enrollment/recovery mechanics
exact IDS transport certificate profile
exact transport framing/protocol
exact queue/backpressure thresholds
exact inspection coverage metrics
exact findings schema
exact retained-context limits
whether full decrypted-session retention is ever supported
exact inspection ZFS dataset/key hierarchy
QUIC/HTTP3 handling
mTLS service-specific proxy mechanisms
inline IPS enforcement architecture
Pathfinder-to-IDS correlation schema
Pathfinder-driven policy vocabulary
fail-open/fail-closed policy vocabulary
performance/capacity profiles
physical HISTORY vs INSPECTION interface requirements
```

None of these decisions are pulled into Phase 0.
