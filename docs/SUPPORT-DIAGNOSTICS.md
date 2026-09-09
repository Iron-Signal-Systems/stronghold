# Stronghold Support, Diagnostics, and Vendor Hardware Access Architecture

## Purpose

This document defines the current architectural direction for Stronghold troubleshooting, diagnostic collection, exceptional engineering access, support bundles, remote-support boundaries, and vendor out-of-band hardware-management preference.

The governing principle is:

> **Stronghold must permit deep troubleshooting without creating an undocumented administrative bypass or a permanent vendor backdoor.**

Stronghold support should make difficult failures diagnosable while preserving configuration authority, identity, capture priority, retention/hold controls, and customer control.

## Troubleshooting Levels

Stronghold separates progressively more privileged support activity:

```text
LEVEL 1 — NORMAL DIAGNOSTICS
    supported CLI/API inspection
    preferably read-only

LEVEL 2 — CONTROLLED SUPPORT COLLECTION
    support bundles
    targeted tests
    deeper platform state
    explicit operator action

LEVEL 3 — ENGINEERING / BREAK-GLASS ACCESS
    exceptional native platform access
    highly privileged
    explicit, attributable, journaled
```

Most operational problems should be solvable without Level 3 access.

## Normal Diagnostics First

Stronghold should expose structured supported diagnostics before requiring native Linux/FreeBSD shell access.

Conceptual families include:

```text
diagnose capture
diagnose interface
diagnose storage
diagnose hunter
diagnose journal
diagnose time
diagnose trust
diagnose ha
diagnose route
diagnose policy
diagnose platform
```

Diagnostics should prefer answering:

```text
what is expected?
what is observed?
what differs?
what responsibility is affected?
what is known?
what remains unknown?
what operator action is indicated?
```

The native platform may supply the measurements, but the supported Stronghold diagnostic surface should correlate them into product meaning where possible.

## Diagnose, Test, Repair

Stronghold preserves clear behavioral verbs:

```text
SHOW
    read state

DIAGNOSE
    inspect/correlate without intended change

TEST
    actively exercise a service/path

REPAIR
    intentionally modify state
```

A diagnostic operation must not silently make undocumented repairs.

High-impact actions remain explicit and separately authorized.

## Support Bundles

Stronghold may create structured support bundles, but a bundle is not an unrestricted filesystem archive.

A support bundle should use a defined manifest and sensitivity-aware inclusion policy.

Typical non-secret diagnostic content may include:

```text
Stronghold release/schema context
kernel / FreeBSD version
hardware/qualification context
service state
interface and driver/firmware state
route/VLAN/bridge summaries
capture counters/ring/drop state
segment/write health
filesystem/ZFS/device health
HA state/synchronization
Hunter backlog/coverage
journal health/checkpoint state
time state
certificate metadata/trust state
bounded diagnostic logs
```

## Sensitivity Classes

Support collection should distinguish data classes conceptually such as:

```text
ordinary health/status
platform/network diagnostic detail
sanitized configuration
sensitive configuration metadata
packet/history content
secret/private credential material
crash/core/memory material
```

Exact class names/numbers are not frozen.

The principle is hard:

> **PCAP, authentication secrets, private keys, and crash/memory dumps are never silently included in an ordinary support bundle.**

## Configuration Sanitization

Support may require configuration context, but secret fields are handled structurally rather than relying on manual redaction.

Example:

```text
radius_server: 10.0.0.8
radius_secret: REDACTED / configured
private_key:   NOT_EXPORTED
```

Support representations should identify whether they are full, sanitized, or redacted and which redaction profile was applied.

This reinforces the requirement that Stronghold configuration references structured secret identities rather than scattering plaintext credentials through free-form files.

## Explicit PCAP Inclusion

Where packet data is required for support, the operator deliberately selects the scope.

A support PCAP extract should preserve provenance such as:

```text
source segment IDs
origin Appliance ID
time range
interface(s)
filter/scope
generation time
export/bundle identity
```

The support copy is a derived export and is not the authoritative source segment.

```text
support PCAP
!=
authoritative source PCAP
```

Stronghold should usually prefer retrieving an extract from existing authoritative capture rather than creating a second invisible capture path.

If an independent diagnostic capture is needed to compare capture behavior, it is explicitly identified as diagnostic and must not be confused with authoritative capture.

## Support Bundle Manifest

Every bundle should identify at least conceptually:

```text
Bundle ID
source Appliance ID
Cluster ID where applicable
Stronghold release
creation time
creator/actor
reason/reference
included components
excluded sensitive classes
redaction profile
file list
hashes/integrity
optional encryption recipient
```

The operator should be able to review what will leave the appliance before export/transmission.

## Bundle Integrity and Encryption

Support bundles should be hash-verifiable and may be signed by the appliance or otherwise carry integrity metadata.

Sensitive bundles should support explicit encryption to a customer-selected or approved support recipient.

Creating, encrypting, and transmitting a support bundle are separate actions.

```text
support bundle created
!=
support bundle transmitted
```

## No Automatic Vendor Upload

> **Stronghold does not automatically transmit support bundles, PCAP, configuration, journals, crash dumps, or telemetry to Iron Signal Systems.**

Any support export/upload is an explicit operator-visible action with a known scope.

This is especially important for government and isolated-network deployments.

## Remote Support

Stronghold has no permanent vendor support account, hidden SSH key, hidden VPN, persistent reverse shell, or always-on vendor tunnel.

If remote support is ever implemented, it is:

```text
disabled by default
customer/operator initiated
time bounded
explicitly scoped
strongly authenticated
visible while active
revocable immediately
management-path constrained
journaled
```

Remote support does not automatically grant firewall configuration, retention, destruction, key-recovery, HA, or other administrative authority.

Potential support privileges remain distinct from broader administrative privileges.

```text
support access
!=
configuration authority

support access
!=
history-destruction authority
```

## Support Identity

Support personnel act under attributable support identities. They do not silently impersonate a customer administrator.

Relevant support events preserve the actor identity, authorization, source, scope, start/end time, and result where applicable.

## Native Engineering Access

There will be exceptional cases where supported diagnostics are insufficient and native platform tools are required.

Stronghold may therefore expose an explicit engineering/break-glass path to Linux/FreeBSD native tooling.

This is not normal product administration.

Conceptual workflow:

```text
Stronghold CLI / console
        ↓
explicit engineering-access request
        ↓
warning + reason + authorization
        ↓
time-bounded native access
        ↓
exit
        ↓
drift / qualification validation
```

Native modification may bypass Stronghold configuration machinery. Stronghold must not pretend that an engineering session preserved the committed configuration state merely because the shell exited successfully.

```text
engineering shell exited
!=
platform state validated
```

## Drift After Native Modification

If native state differs materially from the committed Stronghold generation, Stronghold exposes drift/mismatch rather than silently importing the change.

Conceptual states may include:

```text
NATIVE_ADMIN_SESSION_USED
CONFIGURATION_DRIFT_CHECK_REQUIRED
CONFIGURATION_DRIFT
PLATFORM_QUALIFICATION_MISMATCH
```

The operator may restore the committed Stronghold state or deliberately reproduce the desired change through the Stronghold candidate/commit workflow.

Unmanaged platform modification may place the appliance outside its qualified state until restored/validated, but root access does not automatically make the appliance permanently unsupported.

## Support Activity and Capture Priority

Support and diagnostic work remains secondary to live packet acquisition and durable capture.

Operations such as:

```text
support-bundle compression
debug tracing
diagnostic capture
deep platform scans
```

must throttle, pause, or fail rather than silently create capture loss.

> **Troubleshooting must not create the packet-loss condition it is trying to diagnose without explicitly reporting that impact.**

## Temporary Debug Logging

Higher diagnostic verbosity should be bounded by duration and/or size where practical.

Example conceptual request:

```text
component: capture-engine
debug duration: 15 minutes
maximum diagnostic storage: 1 GB
```

Stronghold should automatically return to normal verbosity rather than allowing temporary debug to become an indefinite storage hazard.

Diagnostic logs remain distinct from authoritative journals.

## Crash and Memory Material

Crash/core dumps and memory dumps may contain:

```text
credentials
session keys
packet payload
private information
authentication tokens
```

They are high-sensitivity support artifacts, not ordinary logs, and require explicit privileged export/handling.

They should never be silently uploaded or included in a standard bundle.

## Support Journaling

Support activity maps to the existing journal domains.

Administrative Journal examples:

```text
support bundle requested/created/exported
engineering access entered/exited
debug level changed
remote support enabled/terminated
```

System/Health examples:

```text
diagnostic collection failed
diagnostic work throttled for capture protection
platform drift detected
```

Trust/Identity examples:

```text
temporary support credential issued/expired/revoked
support identity authenticated
authorization changed
```

A later success does not erase a failed support operation.

## Support Permissions

Support-related privileges are separate from ordinary configuration/destruction authorities.

Conceptual directions may include:

```text
support.bundle.create
support.bundle.include_sensitive
support.remote.enable
support.engineering_access
```

Exact RBAC names remain to be frozen.

Granting support access never implicitly grants:

```text
configuration.commit
history.destroy
retention.admin
hold.admin
key.recover
HA administrative action
```

## Local / Isolated Troubleshooting

Stronghold remains diagnosable when:

```text
Internet unavailable
DNS unavailable
AD unavailable
Hunter unavailable
network isolated
```

A cloud support service is not required merely to create diagnostics or operate the recovery console.

## Vendor Hardware Management Preference

Stronghold manages the appliance. It does not attempt to replace the hardware vendor's supported out-of-band management platform.

> **When access beneath the Stronghold appliance layer is required, Stronghold follows the qualified hardware vendor's preferred/supported management platform and procedures where practical. Stronghold remains responsible for validating the resulting hardware state against the Stronghold-qualified appliance profile.**

Examples of vendor-managed functions may include:

```text
power control
remote console
virtual media
POST visibility
hardware inventory
hardware event logs
firmware management
BIOS/UEFI configuration
storage/controller diagnostics
sensor/thermal state
```

The exact platform depends on the qualified hardware vendor and system.

## Vendor OOB vs Stronghold Management

These are distinct trust domains:

```text
STRONGHOLD MANAGEMENT
    appliance configuration/operation

VENDOR BMC / OOB MANAGEMENT
    physical hardware administration
```

BMC/OOB administration does not become Stronghold configuration authority, and Stronghold administrative permission does not automatically grant BMC authority.

Vendor OOB access normally belongs on a suitable hardware-management network and must not use PRODUCTION, HISTORY, or HA as a hidden management bypass.

```text
BMC administrator
!=
Stronghold administrator
```

## Vendor Support vs Stronghold Qualification

Vendor supportability and Stronghold qualification remain distinct.

```text
vendor-supported hardware
!=
Stronghold-qualified hardware

vendor-supported firmware
!=
Stronghold-qualified firmware
```

A vendor-supported firmware release may still be `NOT_YET_QUALIFIED` for a particular Stronghold release/profile until Stronghold regression testing is complete.

After a vendor-side BIOS/firmware/hardware change, Stronghold validates the resulting platform state and may report `VERIFIED`, `DEGRADED`, `UNQUALIFIED`, `MISMATCH`, or related platform-trust/qualification state as later frozen.

## Recovery Through Vendor OOB

When Stronghold itself cannot boot or expose its management plane, the normal lower-level remote path should be the vendor-supported OOB system where available:

```text
vendor OOB management
        ↓
remote console / virtual media
        ↓
signed Stronghold install/recovery media
        ↓
Stronghold RECOVERY state
```

This is preferable to embedding an always-running hidden Stronghold rescue agent beneath the operating system.

## Support Artifact Retention

Generated support bundles and temporary support workspaces have their own bounded lifecycle.

Deleting an exported support artifact does not delete or rewrite the authoritative history/configuration from which it was derived.

```text
support artifact destroyed
!=
authoritative source destroyed
```

## Support Invariants

> **Stronghold has no permanent vendor backdoor, hidden support account, hidden VPN, persistent vendor SSH key, or automatic remote-support tunnel.**

> **Normal diagnostics are structured and preferably read-only; native engineering/root access is exceptional, explicit, attributable, and followed by drift/qualification validation.**

> **Support bundles have explicit manifests and sensitivity-aware inclusion. PCAP, secrets, private keys, and memory/core dumps are never silently included.**

> **Remote support, if implemented, is customer initiated, time bounded, scoped, visible, attributable, and revocable.**

> **Support activity cannot silently bypass configuration authority, retention/hold controls, identity controls, or capture resource priority.**

> **No Stronghold support artifact or telemetry is automatically transmitted to Iron Signal Systems without explicit customer action.**

> **For access below the appliance layer, use the hardware vendor's supported OOB/management path where practical; Stronghold validates the resulting platform but does not replace the vendor BMC.**

## Truth Separations

```text
diagnose                        != repair
support access                  != configuration authority
support access                  != history-destruction authority
native root change              != Stronghold configuration commit
engineering shell exited        != platform state validated
sanitized config                != full configuration
support PCAP                    != authoritative source segment
support bundle created          != support bundle transmitted
debug log                       != authoritative journal
support artifact deleted        != source history destroyed
Stronghold administration       != BMC administration
vendor-supported hardware       != Stronghold-qualified hardware
vendor-supported firmware       != Stronghold-qualified firmware
BMC access                      != Stronghold configuration authority
hardware change completed       != Stronghold platform validation passed
vendor OOB management           != Stronghold vendor backdoor
```

## Scope

This document defines architecture only. It does not pull a support-bundle implementation, native engineering shell, remote-support system, vendor-BMC integration, crash-dump system, or diagnostic framework into Phase 0.