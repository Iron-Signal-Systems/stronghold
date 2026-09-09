# Stronghold Management Plane Architecture

## Purpose

This document defines the current architectural direction for Stronghold administration across CLI, API, future web management, local console, configuration authority, operational state, and historical state.

The governing principle is:

> **Stronghold has one management authority and multiple interfaces to it. CLI, API, and UI are clients of the same configuration, validation, authorization, and operational-state machinery.**

No management surface owns a separate configuration truth or bypasses Stronghold configuration generation, validation, authorization, or journaling.

## One Management Authority

```text
CLI ─────┐
         │
API ─────┼──► STRONGHOLD MANAGEMENT AUTHORITY
         │            │
UI ──────┘            ├── candidate configuration
                      ├── validation
                      ├── authorization
                      ├── diff
                      ├── commit
                      ├── rollback
                      └── journals
```

The management surfaces express intent. Stronghold owns appliance configuration and the platform state generated from it.

Stronghold must not create separate management paths where, for example, the CLI edits nftables directly, the UI edits a database, and the API edits a different file representation.

## Configuration State

The existing configuration model remains authoritative:

```text
RUNNING
   ↓ copy
CANDIDATE
   ↓ edit
VALIDATE
   ↓
SHOW DIFF
   ↓
COMMIT
   ↓
NEW MONOTONIC GENERATION
```

Rollback restores prior content by creating a new generation. History does not move backward.

Every management surface uses the same candidate, validation, diff, commit, commit-confirmed, rollback, authorization, and journaling contracts.

## Native Platform State

Linux and FreeBSD remain implementation layers beneath Stronghold.

```text
Stronghold configuration
        ↓
validated platform realization
        ↓
Linux / FreeBSD native mechanisms
```

Normal product administration must not require direct editing of nftables, routing tables, bridge state, system service files, FreeBSD rc configuration, PF, ZFS properties, or jail configuration outside the Stronghold management authority.

Native platform changes performed outside Stronghold are not silently imported as product configuration. Stronghold should detect and surface material divergence as configuration/platform drift.

```text
native OS state changed
!=
Stronghold configuration committed
```

## Management State Classes

Stronghold separates:

```text
CONFIGURATION STATE
    what Stronghold is intended to do

OPERATIONAL STATE
    what Stronghold is doing now

HISTORICAL STATE
    what Stronghold did previously
```

Example:

```text
CONFIGURATION
    WAN1 preferred for a destination/service

OPERATIONAL
    WAN2 currently selected because WAN1 is degraded for that target

HISTORICAL
    WAN transition occurred at time T under configuration generation G
```

No management surface should blur these classes merely for display convenience.

## CLI

Stronghold is CLI-first. The CLI is a first-class supported administrative surface rather than a troubleshooting shell hidden behind a GUI.

Expected conceptual command families include:

```text
show
configure
commit
rollback
diagnose
test
repair
journal
history
system
ha
```

Exact syntax is not frozen.

Operational inspection should favor explanation over raw platform dumps. For example, a route inspection should be able to explain the selected prefix/source/next hop and why competing candidates were not selected.

## Local Console vs Remote Management

Local console and remote management are distinct trust paths.

### Local console

The local console is the recovery/break-glass path and may expose controlled capabilities needed for:

```text
local recovery identity
hardware/interface mapping
configuration restore
identity recovery
storage recovery
platform validation
```

External identity-provider failure must not permanently prevent authorized recovery from the local console.

### Remote management

Normal remote management uses configured Stronghold authentication and authorization, including the established directions for LDAPS-only Active Directory, RADIUS, TACACS+, RBAC, and MFA where configured.

Remote management is normally bound to the `MANAGEMENT` role/interface rather than PRODUCTION, HISTORY, or HA paths.

## Native Root / Engineering Access

A generic unrestricted root shell is not the normal Stronghold administration interface.

Native engineering access may exist as an exceptional support/recovery capability under `docs/SUPPORT-DIAGNOSTICS.md`, but native commands do not constitute Stronghold configuration commits.

After native modification, Stronghold must validate drift/qualification rather than assuming the committed generation still describes effective platform state.

## API

The API exposes the same management contract as CLI/future UI. It does not receive a special bypass around validation or authorization.

Conceptual operations include:

```text
read running configuration
read operational state
create/edit candidate
validate candidate
show candidate diff
commit candidate
rollback generation
read journals/history
read capture/Hunter/HA/storage/system state
```

RBAC remains granular. Permission to edit a candidate does not imply permission to commit it.

## API and Object Identity

Automation requires stable object identities independent of mutable names/descriptions.

Examples include:

```text
Policy ID
Interface ID
VLAN object ID
Zone ID
Route ID
WAN preference ID
Appliance ID
Cluster ID
```

Renaming an object must not destroy historical correlation or silently break automation that uses stable identity.

API version, configuration schema version, Stronghold software version, journal schema version, and HA protocol/state format are separate compatibility dimensions.

```text
API version
!=
configuration schema version
!=
Stronghold release version
```

## Future FW Management UI

A future Stronghold FW web UI is another client of the same management authority.

```text
FW MANAGEMENT UI
        ↓
Stronghold management API/authority
        ↓
candidate → validate → diff → commit
```

The UI must not contain a separate privileged configuration backend.

## Net-Hunter External UI Boundary

The Net-Hunter External User Interface Jail remains primarily an investigation surface:

```text
hunt
search
query
timeline
PCAP retrieval/export
reports
status
```

It remains read-only toward authoritative traffic/history and does not gain firewall configuration authority merely because both appliances are part of Stronghold.

```text
Hunter UI access
!=
FW administration authority
```

Common identity providers may be used by both appliances, but an authenticated Hunter session is not automatically a Stronghold FW administrative session.

## Management Service Exposure

Management services are explicitly bound to approved interfaces/roles. Interface existence does not imply SSH/API/UI exposure.

Conceptual direction:

```text
MANAGEMENT
    CLI / SSH-equivalent management
    HTTPS API / future FW UI

PRODUCTION
    no management service by default

HISTORY
    Stronghold FW↔Hunter history/control only

HA
    HA protocol/control only
```

Exceptions require deliberate configuration and remain subject to appliance-local default-deny policy.

## Concurrent Configuration

Stronghold should avoid blindly applying stale candidates.

Example:

```text
running generation = 412
Admin A candidate based on 412
Admin B candidate based on 412
```

If Admin A commits generation 413, Admin B's candidate becomes stale and must be reconciled/rebased before commit.

```text
candidate valid against generation 412
!=
candidate valid against current generation 413
```

Exact concurrency mechanics remain implementation work.

## Administrative Attribution

Configuration-changing operations preserve the responsible actor and management origin where applicable:

```text
user/service identity
authentication source
role/permissions
source address
management interface
surface: console / CLI / API / UI
operation ID
candidate base generation
resulting generation
result
```

A management surface does not become a different authority merely because its origin is recorded.

## Service Identities

Stronghold distinguishes:

```text
HUMAN IDENTITY
SERVICE IDENTITY
APPLIANCE IDENTITY
```

Automation uses scoped service identities rather than shared human credentials. Credential rotation and RBAC apply to service identities independently.

## Secret Handling

Ordinary configuration retrieval does not return reusable secret material.

Example:

```text
RADIUS secret: configured
```

rather than the plaintext secret.

Secret setting/replacement is a distinct privileged operation. General configuration references secret identities rather than repeating plaintext secret values.

## Diagnostic vs Repair Operations

Management vocabulary preserves behavioral intent:

```text
SHOW
    read state

DIAGNOSE
    inspect/correlate without intended modification

TEST
    actively exercise a path/service

REPAIR
    intentionally modify state
```

A diagnostic command must not quietly perform undocumented repair.

Dangerous verbs such as destruction, revocation, key recovery, rollback, restore, failover, and update remain explicit and separately authorized.

## Management Invariants

> **CLI, API, and any future FW management UI are interfaces to one Stronghold management authority.**

> **Native Linux/FreeBSD state is implementation state beneath Stronghold, not a competing supported configuration interface.**

> **Configuration state, operational state, and historical state remain distinct.**

> **Net-Hunter investigation access does not grant firewall configuration authority.**

> **Authentication, authorization, secret handling, validation, generation, and journaling apply consistently regardless of management surface.**

## Truth Separations

```text
CLI configuration                 != separate configuration authority
native OS change                  != Stronghold commit
configured state                  != operational state
operational state                 != historical state
API access                        != commit permission
same identity provider            != same active session
Hunter UI access                  != FW administration authority
diagnostic action                 != repair action
candidate valid on old generation != candidate valid on current generation
secret configured                 != secret readable
```

## Scope

This document defines architecture only. It does not pull a complete CLI, API, FW web UI, management service, remote authentication implementation, or management concurrency system into Phase 0.