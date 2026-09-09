# Stronghold IDS / TLS Inspection Architecture

## Purpose

This document defines the current future architecture for Stronghold TLS/application inspection and IDS/IPS integration.

**IDS/IPS implementation remains deferred.** This document establishes the architectural boundaries that any later implementation must preserve. It does not select an IDS engine, ruleset format, final proxy implementation, or complete IPS enforcement contract.

The governing principle is:

> **Useful inspection of encrypted application traffic requires plaintext somewhere, but decrypted traffic must not become a second uncontrolled historical datastore or a mandatory dependency for ordinary Stronghold forwarding.**

Stronghold therefore separates authoritative wire observation, selective proxy decryption, transient IDS transport, and persisted detection findings.

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
```

The following distinctions are mandatory:

```text
original encrypted PCAP        != decrypted inspection history
IDS finding                    != authoritative packet
IDS finding retained           != full decrypted session retained
inspection feed lost           != packet observation lost
decrypted in FW memory         != decrypted retained on FW
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

Examples:

```text
Inspection: PROXY_INSPECTED
Reason: policy INSPECT-USERS-WEB

Inspection: BYPASSED
Reason: PINNED_APPLICATION

Inspection: BYPASSED
Reason: MUTUAL_TLS

Inspection: NOT_AVAILABLE
Reason: UNSUPPORTED_PROTOCOL
```

Inspection bypass never means the original traffic was not observed by the Stronghold physical-interface capture path.

## Inspection CA and Trust Separation

The inspection certificate authority is purpose-specific.

Stronghold should favor a customer-controlled inspection trust architecture rather than any universal Iron Signal Systems interception root.

Conceptually:

```text
CUSTOMER TRUST / PKI
        │
        └── STRONGHOLD INSPECTION CA
                ├── temporary leaf for requested hostname A
                ├── temporary leaf for requested hostname B
                └── ...
```

Inspection-signing credentials remain separate from:

```text
Stronghold appliance identity
FW↔Hunter HISTORY transport identity
HA identity
management/API identity
journal-signing identity
IDS transport identity
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

The later implementation should use bounded process buffers and platform-appropriate memory/dump controls. A memory filesystem must not be assumed to satisfy the contract merely because it is named `tmpfs` or equivalent.

Stronghold support/diagnostic tooling must treat inspection memory and crash artifacts as high-sensitivity material.

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

Both sides require:

```text
certificate validity
+
correct IDS/inspection credential purpose
+
explicit peer authorization
```

A valid certificate alone is never sufficient authorization.

## IDS Transport Identity Separation

IDS/inspection transport credentials are separate from authoritative HISTORY transport credentials.

Conceptually:

```text
FW Appliance ID
    ├── HISTORY transport credential
    └── INSPECTION transport credential

Hunter Appliance ID
    ├── HISTORY ingest credential
    └── IDS inspection credential
```

The exact certificate profile, extension/purpose representation, enrollment, rotation, and revocation mechanics remain to be frozen with the Stronghold cryptographic profile.

```text
HISTORY transport identity
!=
INSPECTION transport identity
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

They require separate:

```text
credentials
authorization
queues
resource accounting
health state
failure state
```

A later high-throughput profile may qualify a dedicated physical `INSPECTION` interface if measurement shows that sharing the history fabric risks authoritative transfer or predictable inspection service.

## Net-Hunter IDS / Inspection Jail

The future IDS function belongs in a distinct isolated jail rather than being silently folded into the existing Record Processing Jail.

The current conceptual jail set becomes:

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

    inspection/
      findings/
      retained-context/
```

The exact dataset names are illustrative, not frozen.

The `inspection` domain is encrypted at rest and may use a separate encryption root/key from ordinary authoritative and derived datasets.

The **FreeBSD host retains ZFS/key-management authority**. The IDS jail receives only the mounted dataset access required for its role.

```text
IDS jail access
!=
ZFS key-management authority
```

## Inspection Persistence Classes

Stronghold should preserve multiple explicit persistence classes rather than silently retaining full plaintext whenever IDS is enabled.

Conceptual direction:

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
```

A finding is derived interpretation and never replaces the authoritative encrypted-wire PCAP.

Because findings themselves may contain sensitive information such as URLs, hostnames, headers, filenames, usernames, payload fragments, or malware strings, the findings store is encrypted at rest even when full decrypted sessions are not retained.

## Historical Reprocessing Limitation

If Stronghold does not retain decrypted sessions or TLS session secrets, later reprocessing of old encrypted PCAP cannot reconstruct the historical plaintext merely because the original wire PCAP still exists.

```text
encrypted historical PCAP
!=
historical plaintext available
```

Stronghold should not retain TLS session secrets by default merely to enable later retrospective decryption. Doing so would substantially increase the breach impact of Net-Hunter and requires a separate explicit future security decision if ever considered.

## Inspection Retention

Inspection findings/context have retention independent from authoritative PCAP and other journal domains.

Conceptually:

```text
AUTHORITATIVE PCAP
    retention policy A

IDS FINDINGS
    retention policy B

DECRYPTED RETAINED CONTEXT
    retention policy C
```

Holds, destruction authorization, crypto-shred authority, and journaled destruction apply according to the eventual retention model for each class.

```text
IDS finding expired
!=
authority to destroy authoritative PCAP
```

## IDS Resource Priority

Stronghold FW remains a firewall/capture appliance first.

On FW, the inspection system performs only the work needed to proxy the selected live connection and create the bounded transient inspection feed.

Deep IDS parsing/rule evaluation belongs on Hunter where possible.

The inspection path never outranks:

```text
packet acquisition
active authoritative PCAP writes
essential segment finalization/integrity
essential forwarding/enforcement work
critical HA heartbeat/control
authoritative history transfer needed to protect backlog
```

Exact ordering against other secondary work must be benchmarked later.

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

The exact counter/location/granularity contract remains to be frozen.

A future explicitly configured `MUST_INSPECT` or IPS policy may choose fail-closed behavior, but such behavior must be explicit, scoped, and separately qualified. It is not the global default merely because IDS/IPS exists.

```text
IDS unavailable
!=
production forwarding unavailable
```

unless explicit policy requires that dependency.

## IDS vs IPS

Deep Hunter analysis should not become a per-packet synchronous forwarding oracle.

The architecture intentionally avoids:

```text
packet
  ↓
FW decrypts
  ↓
wait for Hunter deep IDS decision
  ↓
FW forwards/drops
```

for ordinary traffic.

Future IPS may use carefully bounded mechanisms such as:

```text
local FW proxy termination for specifically qualified immediate rules
Hunter detection producing an authorized dynamic enforcement fact for new/future flows
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

Possible future policy behaviors may include:

```text
native QUIC proxy/termination
explicit QUIC bypass
policy-driven HTTP/3 suppression/fallback
```

Stronghold must report which behavior occurred. It must not silently disable QUIC and imply native inspection support.

Exact QUIC architecture remains unfrozen.

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

```text
pinned application
    → BYPASS / UNSUPPORTED according to policy
```

This failure/bypass state remains visible in inspection coverage.

## Inspection Coverage

Stronghold must not present `DPI ENABLED` as proof that encrypted traffic was actually inspected.

Inspection coverage should be able to distinguish at least conceptually:

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
```

An incomplete inspection feed or unsupported application must remain visible as coverage limitation.

## Support and Diagnostic Boundaries

Support bundles, debug logging, crash handling, and engineering access must preserve the support architecture in `docs/SUPPORT-DIAGNOSTICS.md`.

In particular:

- decrypted inspection payload is never silently included in ordinary support bundles;
- IDS memory/core dumps are high-sensitivity artifacts;
- debug logging must not casually dump plaintext payload;
- inspection at-rest keys are not exposed to support identities merely because they can diagnose the jail;
- temporary diagnostic actions must not create a plaintext spool.

## Health and Alerting

The observability architecture in `docs/OBSERVABILITY.md` later needs independent IDS/inspection health such as:

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
```

Healthy forwarding/capture must not hide degraded inspection, and degraded inspection must not be reported as packet-observation loss when the authoritative capture path remained healthy.

## Architecture Invariants

The following are frozen architectural requirements for any later IDS/TLS-inspection implementation:

> **TLS/application inspection is an explicitly selected proxy function, not a prerequisite for ordinary Stronghold forwarding.**

> **An inspected connection remains encrypted on both sides of Stronghold. Plaintext exists only inside controlled volatile processing boundaries unless an explicit encrypted-retention policy permits selected IDS material to persist.**

> **Stronghold FW never intentionally persists decrypted inspection payload at rest.**

> **Stronghold FW transports decrypted inspection material to Net-Hunter only over a dedicated mutually authenticated encrypted inspection channel using purpose-specific IDS/inspection credentials and explicit peer authorization. There is no plaintext fallback.**

> **Net-Hunter IDS processing occurs in an isolated future IDS/Inspection Jail. Any inspection-derived material that persists is stored only in a dedicated encrypted-at-rest inspection domain whose key-management authority remains with the Hunter host.**

> **The INSPECTION transport, identities, queues, health, persistence, and failure semantics remain separate from authoritative HISTORY transport.**

> **Normal IDS overload or Hunter unavailability does not silently backpressure production forwarding. Inspection loss/degradation is measured and reported. Any fail-closed behavior must be explicit and scoped.**

> **Full decrypted sessions and TLS session secrets are not retained by default. Findings-only is the default architectural direction; bounded/full decrypted retention requires explicit later policy and qualification.**

> **Authoritative encrypted PCAP remains the wire-history authority. IDS findings are derived interpretation and retain provenance toward the applicable original history where available.**

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
fail-open/fail-closed policy vocabulary
performance/capacity profiles
physical HISTORY vs INSPECTION interface requirements
```

None of these decisions are pulled into Phase 0.