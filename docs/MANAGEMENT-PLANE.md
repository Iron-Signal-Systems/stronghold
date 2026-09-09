# Stronghold Management Plane Architecture

## Purpose

This document defines the current architectural direction for Stronghold administration across CLI, API, future web management, local console, configuration authority, operational state, and historical state.

The governing principle is:

> **Stronghold has one management authority and multiple interfaces to it. CLI, API, and UI are clients of the same configuration, validation, authorization, and operational-state machinery.**

No management surface owns a separate configuration truth or bypasses Stronghold configuration generation, validation, authorization, Git-backed lineage, mandatory change attribution, or journaling.

Detailed governing contracts:

- `docs/CONFIGURATION-GOVERNANCE.md` — Git-backed configuration lineage, mandatory change comments, least privilege, administrative authorization, approval/separation-of-duties direction, and historical troubleshooting.
- `docs/STATEFUL-ENFORCEMENT.md` — live-session authorization, policy-generation reconciliation, rule move/remove behavior, session rebinding/termination, and post-commit runtime impact.
- `docs/POLICY-SIMULATION.md` — non-mutating Dry Run/counterfactual testing, candidate impact estimation, IPv4/IPv6 and FQDN family-aware simulation, session/route/NAT/WAN impact prediction, and expected-versus-actual comparison.

## One Management Authority

```text
CLI ─────┐
         │
API ─────┼──► STRONGHOLD MANAGEMENT AUTHORITY
         │            │
UI ──────┘            ├── candidate configuration
                      ├── validation
                      ├── diff
                      ├── Dry Run / simulation
                      ├── expected-impact review
                      ├── authorization / approval
                      ├── mandatory change comment
                      ├── Git-backed finalization
                      ├── generation / activation
                      ├── runtime reconciliation
                      ├── post-commit impact
                      ├── rollback
                      └── journals / history
```

The management surfaces express intent. Stronghold owns appliance configuration and the platform state generated from it.

Stronghold must not create separate management paths where, for example, the CLI edits nftables directly, the UI edits a database, and the API edits a different file representation.

## Configuration State

The configuration model is:

```text
RUNNING
   ↓ copy
CANDIDATE
   ↓ edit
VALIDATE
   ↓
SHOW DIFF
   ↓
DRY RUN / SIMULATE
   ↓
REVIEW EXPECTED IMPACT
   ↓
SAVE / COMMIT REQUEST
   ↓
MANDATORY CHANGE COMMENT
   ↓
AUTHORIZATION / REQUIRED APPROVAL
   ↓
GIT COMMIT
   ↓
NEW MONOTONIC GENERATION
   ↓
ACTIVATE
   ↓
RUNTIME RECONCILIATION
   ↓
POST-COMMIT OBSERVATION
```

A candidate is temporary working state. Candidate editing, validation, diff, simulation, and discard do not themselves change durable Stronghold configuration.

Dry Run is a non-mutating management operation governed by `docs/POLICY-SIMULATION.md`. It may evaluate a candidate against hypothetical flows, current sessions, policy ordering, object dependencies, routing, NAT, WAN behavior, FQDN resolution, and other supported facts without making the candidate authoritative or changing production runtime state.

A simulation does not require a configuration-change comment merely to run because it is not a durable configuration change. Simulation remains separately authorized because its output may expose sensitive network, policy, route, DNS, and session information.

```text
simulation authorized
!=
configuration finalization authorized
```

When an operator invokes the finalizing action — whatever production vocabulary is later selected, such as `save`, `write`, `commit`, or equivalent — a change comment is mandatory.

```text
no comment
    =
no save
    =
no Git commit
    =
no new generation
    =
no runtime configuration change
```

All durable configuration is Git-backed, including administratively disabled/inactive objects. A disabled rule is still configuration and therefore has versioned lineage.

Rollback restores prior content by creating a new Git commit and a new Stronghold generation. History does not move backward or erase the mistaken generation.

Every management surface uses the same candidate, validation, diff, Dry Run/simulation, mandatory-comment, authorization, Git-finalization, generation, commit-confirmed, rollback, reconciliation, post-commit impact, and journaling contracts.

## Configuration History Versus Administrative History

Stronghold uses separate but correlated histories:

```text
GIT CONFIGURATION HISTORY
    what configuration content changed

STRONGHOLD ADMINISTRATIVE JOURNAL
    who requested/authorized/activated the change
    how the request entered Stronghold
    why the change was made
    whether validation/activation succeeded
    what Stronghold did afterward
```

Each finalized Stronghold generation resolves to exact versioned configuration content.

```text
Git commit exists
!=
activation succeeded

Git history valid
!=
administrative journal valid
```

Accepted configuration history is forward-moving. Normal Stronghold operation must not rewrite accepted appliance configuration history using reset/rebase/amend/force-push semantics.

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
simulate
repair
journal
history
system
ha
```

Exact syntax is not frozen.

Operational inspection should favor explanation over raw platform dumps. For example, a route inspection should be able to explain the selected prefix/source/next hop and why competing candidates were not selected.

Dry Run/policy simulation should eventually support capabilities such as:

```text
test policy source HR-PC-17 destination payroll.vendor.com service HTTPS
simulate candidate against current sessions
simulate policy P-01872
simulate route changes
simulate NAT changes
show simulation terminated
show simulation restart-required
```

The exact syntax remains implementation work. Simulation output is prediction and must use vocabulary such as `EXPECTED RESULT`, `ESTIMATED IMPACT`, `WOULD ALLOW`, `WOULD DENY`, `WOULD REBIND`, `WOULD RESTART`, and `WOULD TERMINATE` rather than pretending the predicted packet or session event actually occurred.

Historical troubleshooting should eventually support capabilities such as:

```text
show changes since "7 days ago"
show changes user DOMAIN\\tech1
show changes affecting policy P-01872
show diff generation 819 820
show config at 2026-09-02T12:00
show impact generation 820
```

The exact command syntax remains implementation work; the capability to answer **what changed, who changed it, why, and what happened afterward** is the architectural requirement.

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

Break-glass does not become an undocumented unrestricted routine path. Exceptional operations remain attributable, reasoned, authorized according to the available recovery authority, and journaled.

### Remote management

Normal remote management uses configured Stronghold authentication and authorization, including the established directions for LDAPS-only Active Directory, RADIUS, TACACS+, RBAC, and MFA where configured.

Remote management is normally bound to the `MANAGEMENT` role/interface rather than PRODUCTION, HISTORY, or HA paths.

## Native Root / Engineering Access

A generic unrestricted root shell is not the normal Stronghold administration interface.

Native engineering access may exist as an exceptional support/recovery capability under `docs/SUPPORT-DIAGNOSTICS.md`, but native commands do not constitute Stronghold configuration commits.

After native modification, Stronghold must validate drift/qualification rather than assuming the committed generation still describes effective platform state.

## Administrative Authorization and Least Privilege

Stronghold adopts the applicable security concepts from the Iron Signal Systems Domain-Neutral Platform without becoming runtime-dependent on DNP, its database, or its services.

The relevant concepts include:

```text
identity is not authorization
least privilege
scoped authority
exact operation/target context
revocable authority
step-up where required
independent approval where required
separation of duties where required
attributable decisions
fail-closed required authorization stages
```

Stronghold should not define an ordinary day-to-day account or accumulated role set that silently grants unrestricted product authority.

Potential granular operation families include:

```text
policy.create / modify / move / disable / remove / activate
policy.simulate
object.create / modify / remove
nat.modify
route.modify
interface.modify
capture.configure / export
ztna.policy.modify / session.revoke
wireguard.peer.authorize
site_tunnel.modify
ha.configure / promote
trust.certificate.modify / trust.root.modify
update.activate
recovery.execute
history.destroy
```

Exact names and shipped roles remain implementation work.

Stronghold preserves:

```text
user authenticated
!=
operation authorized

role/group membership
!=
unrestricted authority

permission to edit candidate
!=
permission to finalize change

permission to simulate candidate
!=
permission to activate candidate

authorized to request
!=
authorized to approve

authorized to approve
!=
authorized to execute
```

Missing, ambiguous, expired, revoked, superseded, incompatible, or unevaluated required authority fails closed.

High-impact operations may require step-up MFA, independent approval, or separation of duties. Examples include trust-root/signing-authority changes, disabling/degrading capture protection, altering journal-integrity controls, weakening HA fencing, forced promotion, management-plane exposure changes, privileged recovery, destructive history operations, and any future hardware-bypass mode.

## API

The API exposes the same management contract as CLI/future UI. It does not receive a special bypass around validation, authorization, mandatory change comments, Git-backed finalization, generation assignment, or reconciliation.

Conceptual operations include:

```text
read running configuration
read operational state
create/edit candidate
validate candidate
show candidate diff
run non-mutating policy/candidate simulation
read simulation result / expected impact
finalize candidate with mandatory change reason
rollback generation
read journals/history
read capture/Hunter/HA/storage/system state
```

RBAC remains granular. Permission to edit a candidate does not imply permission to finalize it. Permission to run a Dry Run does not imply permission to activate the candidate.

Automation-originated durable changes require the same attributable reason/comment semantics as human changes. The reason may be supplied programmatically but remains mandatory and recorded.

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

Renaming or moving an object must not destroy historical correlation or silently break automation that uses stable identity.

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
candidate → validate → diff → Dry Run / expected impact
        ↓
mandatory comment → authorize → Git commit
        ↓
generation → activate → reconcile → post-commit impact
```

The UI must not contain a separate privileged configuration backend.

A future UI may provide a prominent Dry Run action before activation, but it must not quietly convert Dry Run into a configuration commit or production dataplane test.

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

If Admin A finalizes generation 413, Admin B's candidate becomes stale and must be reconciled/rebased before finalization.

```text
candidate valid against generation 412
!=
candidate valid against current generation 413
```

A simulation is also bound to the candidate/base context used to produce it. A prior Dry Run must be marked stale where Stronghold can establish that the candidate, base generation, or material dependent state changed.

```text
simulation against generation 412
!=
simulation current after generation 413 becomes authoritative
```

A successful simulation never becomes an authorization token for stale candidate activation.

Exact concurrency mechanics remain implementation work.

## Administrative Attribution

Configuration-changing operations preserve the responsible actor and management origin where applicable:

```text
user/service identity
authentication source
device/session identity where applicable
authority/grant used where applicable
approval identity/request where applicable
source address
management interface
surface: console / CLI / API / UI
operation ID
candidate base generation
Git commit ID
previous generation
resulting generation
mandatory change comment
validation result
activation result
runtime reconciliation result
```

A management surface does not become a different authority merely because its origin is recorded.

Non-mutating simulations are also attributable management operations. Their records should preserve a Simulation ID, actor/service identity, candidate/base identity, simulation class, request context, and result status without pretending a configuration generation was created.

## Historical Administrator Review

Authorized senior administrators/auditors should be able to filter configuration and privileged administrative history by actor, time range, stable object, management surface, and operation class.

Potential capabilities include:

```text
changes by actor
privileged actions by actor
denied administrative actions by actor
authority exercised by actor
approvals by actor
simulations by actor
changes affecting a Policy ID / VLAN / route / object
changes within a selected time range
```

This is a troubleshooting and accountability capability, not an assumption that the selected technician made an error.

## Pre-Commit Dry Run and Expected Impact

Before durable finalization, Stronghold should allow an authorized operator to evaluate likely candidate effects without altering production.

The detailed contract is governed by `docs/POLICY-SIMULATION.md`.

Potential capabilities include:

```text
single-flow policy evaluation
candidate against current sessions
rule move/removal impact
referenced-object impact
route/NAT/WAN impact
IPv4/IPv6 family-aware FQDN evaluation
future secure-access request simulation
```

Dry Run must not terminate sessions, create production state, program nftables/routes/NAT/WireGuard, change HA state, increment production counters, or otherwise alter production behavior.

## Post-Commit Impact

Policy-affecting changes should correlate configuration history with runtime and packet-history consequences.

Conceptually:

```text
ADMINISTRATOR / COMMENT
        ↓
GIT DIFF
        ↓
PRE-COMMIT EXPECTED IMPACT
        ↓
STRONGHOLD GENERATION
        ↓
POLICY / SESSION RECONCILIATION
        ↓
NEW ALLOW / DENY DECISIONS
        ↓
AUTHORITATIVE PACKET HISTORY
        ↓
EXPECTED vs ACTUAL COMPARISON
```

This enables an operator to discover an unintended rule movement/removal quickly rather than waiting hours for a previously established client session to close and expose the problem.

`docs/STATEFUL-ENFORCEMENT.md` governs the detailed live-session behavior. `docs/POLICY-SIMULATION.md` governs pre-commit prediction and expected-versus-actual comparison.

Post-commit impact reporting is observation/explanation. Stronghold must not automatically declare a syntactically valid change to be correct or incorrect merely because traffic patterns changed.

A particularly important warning is when observed behavior materially differs from the Dry Run estimate:

```text
SIMULATION EXPECTED:
    0 new denies

ACTUAL SINCE COMMIT:
    38 newly denied flows

ATTENTION:
    observed impact differs from simulation
```

This does not automatically roll back or declare the change wrong. It gives the operator immediate, attributable information while the change context is still fresh.

## Service Identities

Stronghold distinguishes:

```text
HUMAN IDENTITY
SERVICE IDENTITY
APPLIANCE IDENTITY
```

Automation uses scoped service identities rather than shared human credentials. Credential rotation and least-privilege authorization apply to service identities independently.

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

TEST / SIMULATE
    evaluate or actively exercise only according to the documented command contract

REPAIR
    intentionally modify state
```

Dry Run/simulation commands governed by `docs/POLICY-SIMULATION.md` are non-mutating. A diagnostic or simulation command must not quietly perform undocumented repair or production configuration.

Dangerous verbs such as destruction, revocation, key recovery, rollback, restore, failover, and update remain explicit and separately authorized.

## Management Invariants

> **CLI, API, and any future FW management UI are interfaces to one Stronghold management authority.**

> **Dry Run is predictive and non-mutating; activation remains the authority that changes Stronghold behavior.**

> **No durable Stronghold configuration change exists without an attributable operator-supplied reason.**

> **Every active Stronghold configuration generation references exact versioned configuration content.**

> **Authentication establishes identity; it does not grant unrestricted administrative authority.**

> **Native Linux/FreeBSD state is implementation state beneath Stronghold, not a competing supported configuration interface.**

> **Configuration state, operational state, and historical state remain distinct.**

> **Net-Hunter investigation access does not grant firewall configuration authority.**

> **Authentication, authorization, secret handling, validation, simulation, Git lineage, generation, reconciliation, and journaling apply consistently regardless of management surface.**

## Truth Separations

```text
CLI configuration                 != separate configuration authority
native OS change                  != Stronghold commit
candidate edited                  != durable configuration changed
simulation performed              != configuration changed
simulation authorized             != activation authorized
simulation predicts ALLOW         != packet actually allowed
simulation match                  != production policy hit
candidate impact estimate         != post-commit observed impact
configured state                  != operational state
operational state                 != historical state
API access                        != finalization permission
user authenticated                != operation authorized
role/group membership             != unrestricted authority
comment supplied                  != operation authorized
Git commit created                != activation succeeded
configuration activated           != reconciliation complete
Git history valid                 != administrative journal valid
rollback                          != history deletion
same identity provider            != same active session
Hunter UI access                  != FW administration authority
diagnostic action                 != repair action
candidate valid on old generation != candidate valid on current generation
secret configured                 != secret readable
```

## Scope

This document defines architecture only. It does not pull a complete CLI, API, FW web UI, Git implementation, authorization/approval service, policy simulator, management concurrency system, historical-query engine, or post-commit impact UI into Phase 0.
