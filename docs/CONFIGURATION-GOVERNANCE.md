# Stronghold Configuration Governance and Administrative Authorization

## Purpose

This document defines the architectural contract for Stronghold configuration finalization, Git-backed configuration lineage, administrative authorization, least privilege, attribution, historical troubleshooting, and change-impact correlation.

The governing rules are:

> **No durable Stronghold configuration change exists without an attributable operator-supplied reason.**

> **Authentication establishes identity; it does not grant unrestricted administrative authority.**

> **Every active Stronghold configuration generation references exact versioned configuration content.**

> **Configuration history is forward-moving and is not rewritten to conceal prior state.**

This model applies across CLI, API, future UI, automation/service identities, and controlled local-console recovery paths according to the authority available on each surface.

## Candidate Versus Durable Configuration

A candidate is temporary working state.

Conceptually:

```text
RUNNING
   ↓ copy
CANDIDATE
   ↓ edit
VALIDATE
   ↓
SHOW DIFF
   ↓
SAVE / COMMIT REQUEST
   ↓
MANDATORY CHANGE COMMENT
   ↓
AUTHORIZATION
   ↓
GIT COMMIT
   ↓
NEW MONOTONIC STRONGHOLD GENERATION
   ↓
ACTIVATE
   ↓
RUNTIME RECONCILIATION
```

Candidate editing, validation, diff, simulation, and discard may occur without creating durable configuration.

Once the operator invokes the finalizing action — whatever production vocabulary is later selected, such as `save`, `write`, `commit`, or equivalent — a change comment is mandatory.

```text
no comment
    =
no save
    =
no Git commit
    =
no new Stronghold generation
    =
no runtime configuration change
```

If the finalization prompt is cancelled, the active configuration remains unchanged. The candidate may remain available for continued editing or explicit discard according to the later CLI/UI contract.

## Mandatory Change Comment

Every durable configuration change requires an operator-supplied comment/reason before Stronghold will finalize it.

The requirement applies regardless of management surface:

```text
CLI
API
automation
future FW UI
controlled local-console configuration
```

An API cannot bypass the rule; the equivalent request field is mandatory.

The exact validation contract remains implementation work, but at minimum the comment must be non-empty/non-whitespace and bounded in length.

Stronghold should not depend on subjective AI judgment to decide whether a comment is acceptable.

Examples of useful comments:

```text
CHG-2871 permit new payroll vendor HTTPS
Move HR payroll allow before general internet deny
INC-2197 restore known-good HR policy after rule-order regression
Replace expiring enterprise CA trust root
```

A ticket/reference may be recorded separately where an organization uses formal change management, but Stronghold must not require an external ticketing system as a runtime dependency.

## Git-Backed Configuration Lineage

All durable Stronghold configuration is versioned in a Stronghold-controlled Git repository or equivalent Git-backed configuration store.

This includes configuration objects that are administratively disabled/inactive. `disabled` is configuration content and is therefore versioned.

Each finalized Stronghold generation must resolve to exact versioned configuration content.

Conceptually:

```text
Git commit 7f29...  <->  Stronghold generation 841
Git commit d72a...  <->  Stronghold generation 842
Git commit a813...  <->  Stronghold generation 843
```

Git provides configuration-content lineage. Stronghold journals provide authoritative administrative/action history.

```text
Git history
    answers: what configuration content changed?

Stronghold Administrative Journal
    answers: who requested/authorized/activated it, from where, why,
             whether it succeeded, and what Stronghold did afterward?
```

Therefore:

```text
Git commit exists
!=
activation succeeded

Git history valid
!=
administrative journal valid
```

## No Routine History Rewriting

Normal Stronghold configuration operation must not use history-rewriting workflows such as:

```text
git reset --hard to erase accepted history
git rebase of accepted appliance configuration history
git commit --amend of accepted appliance configuration history
force-push of accepted appliance configuration history
```

Rollback restores older content by creating a new Git commit and a new Stronghold generation.

Example:

```text
841  known-good configuration
842  mistaken change
843  restore content based on 841
```

Generation 842 remains visible forever according to the applicable retention/history contract.

```text
rollback
!=
history deletion
```

## Administrative Attribution

Every finalized configuration change preserves enough context to establish who performed it and how.

Potential fields include:

```text
operation/change ID
Git commit ID
candidate base generation
previous active generation
new Stronghold generation
human or service identity
authentication source
authentication/session ID where applicable
device identity where applicable
authority/grant used where applicable
approval/request IDs where applicable
source address
management interface
management surface: console / CLI / API / UI
timestamp / authoritative time context
mandatory change comment
validation result
activation result
runtime reconciliation result
```

Exact schema remains implementation work.

## DNP-Inspired Authority Model

Stronghold adopts the security concepts established in the Iron Signal Systems Domain-Neutral Platform where they fit the Stronghold appliance threat model, including:

```text
identity is not authorization
least privilege
scoped authority
exact operation/target context
revocable authority
separation of duties
independent approval where required
short-lived authorization where appropriate
attributable decisions
fail-closed required stages
```

Stronghold does not become runtime-dependent on DNP, its database, or its services. Stronghold implements these concepts independently within the appliance boundary.

Conceptual reuse of the model does not imply shared state or a hidden external authorization dependency.

## No Unrestricted Ordinary Administrative Account

Stronghold should not define an ordinary day-to-day account or accumulated role set that silently grants unrestricted product authority.

Administrative authority is scoped to governed operations and targets.

Potential operation families include:

```text
policy.create
policy.modify
policy.move
policy.disable
policy.remove
policy.activate

object.create
object.modify
object.remove

nat.modify
route.modify
interface.modify
vlan.modify
zone.modify

capture.configure
capture.export

ztna.policy.modify
ztna.session.revoke
wireguard.peer.authorize
site_tunnel.modify

ha.configure
ha.promote

trust.certificate.modify
trust.root.modify

update.activate
recovery.execute
history.destroy
```

Exact operation names remain to be frozen later.

Authority may also be scoped by appliance/cluster, interface/zone, object family, resource target, organization, effective period, or other Stronghold context where justified.

## Authentication Is Not Authorization

Stronghold preserves:

```text
user authenticated
!=
operation authorized

role/group membership
!=
unrestricted authority

authorized to edit candidate
!=
authorized to finalize change

authorized to modify policy
!=
authorized to modify trust

authorized to request
!=
authorized to approve

authorized to approve
!=
authorized to execute
```

A missing, ambiguous, expired, revoked, superseded, incompatible, or unevaluated required authority condition fails closed.

Stronghold must not fall back to broad administrative permission merely because an identity is generally considered an administrator.

## Least-Privilege Examples

A Network Technician might be authorized to:

```text
create/edit host and network objects
modify approved VLANs/interfaces
create/edit candidate security policy
run policy simulation
view permitted operational/capture history
```

while lacking authority to:

```text
modify trust roots
change HA fencing/election
alter journal integrity configuration
disable capture
change appliance update trust
perform privileged recovery
```

A Security Administrator may have broader policy/NAT/ZTNA authority while still lacking trust-root or recovery authority.

A Senior Network Administrator may have wider scope while high-impact operations remain separately governed.

The exact shipped roles are implementation/product-design work; the architecture requires granular authority rather than a single unrestricted firewall-admin capability.

## Step-Up and Independent Approval

Routine configuration should remain operationally practical. Not every ordinary rule edit requires two-person approval.

High-impact operations may require stronger control such as step-up MFA, exact-context authorization, independent approval, or separation of duties.

Examples worth governing more strongly include:

```text
disable/degrade capture protection
change trust root or signing authority
alter journal verification/integrity controls
force HA promotion
weaken/disable HA fencing
change management-plane exposure
perform appliance identity recovery
change update-signing authority
execute destructive history/retention operations
enable any future hardware-bypass mode
```

Where independent approval is required, requester and approver identity/authority relationships are evaluated explicitly rather than by simple row/count semantics.

## Stable Object Identity and Historical Lineage

Stronghold object identity is stable independently of mutable name, position, or description.

Historical queries should work against identifiers such as:

```text
Policy ID
Host/Network Object ID
Service Object ID
Interface ID
VLAN Object ID
Zone ID
Route ID
NAT ID
WAN Preference ID
Appliance ID
Cluster ID
Secure-Access Policy/Session ID
```

Renaming or moving an object must not erase its history.

## Historical Troubleshooting

The management plane should allow an authorized operator — particularly senior troubleshooting staff — to ask operational questions such as:

```text
show changes since "7 days ago"
show changes user DOMAIN\\tech1
show changes user DOMAIN\\tech1 since "7 days ago"
show changes affecting policy P-01872
show changes affecting vlan 120
show changes affecting host HR-SERVER-01
show changes affecting 10.40.18.0/24
show diff generation 819 820
show config at 2026-09-02T12:00
show impact generation 820
```

Exact CLI syntax is not frozen; the capability is the architectural requirement.

The goal is to answer:

> **This worked last week. What changed?**

without manually searching arbitrary backup files or relying on administrator memory.

## Administrator-Centric Review

Authorized senior administrators/auditors should be able to filter configuration history and privileged administrative actions by actor.

Potential views include:

```text
changes by actor
privileged actions by actor
denied administrative actions by actor
authority exercised by actor
approvals by actor
changes by management surface
changes within a time range
changes affecting a stable Stronghold object
```

This is an operational investigation capability, not an assumption that the selected technician made an error.

## Post-Commit Operational Impact

Configuration history should correlate with runtime consequences.

For policy-affecting generations, Stronghold should be able to pivot conceptually through:

```text
ADMINISTRATOR
    ↓
MANDATORY CHANGE COMMENT
    ↓
GIT DIFF
    ↓
STRONGHOLD GENERATION
    ↓
POLICY/RUNTIME RECONCILIATION
    ↓
ALLOW / DENY DECISIONS
    ↓
AUTHORITATIVE PACKET HISTORY
```

Potential post-commit output may include:

```text
sessions evaluated
sessions rebound
sessions terminated
newly allowed flows
newly denied flows
top affected sources/destinations/services
first observed impact time
reconciliation status
```

`docs/STATEFUL-ENFORCEMENT.md` governs live policy/session reconciliation behavior.

Near-real-time packet capture makes this operationally valuable: an administrator can see an unintended deny immediately after a rule move/removal rather than learning about it hours later when a previously established client session finally closes.

## Commit-Confirmed

Stronghold may support a `commit-confirmed` or equivalent protection for risky remote changes.

The change still follows the normal rules:

```text
mandatory comment
authorization
Git-backed generation
activation
reconciliation
journaling
```

If confirmation does not occur within the authorized window, Stronghold restores the prior content by creating a new Git commit/new generation. The temporary generation is not erased.

## Management-Surface Consistency

CLI, API, future UI, and automation are clients of one Stronghold management authority.

None may bypass:

```text
candidate semantics
mandatory change comment
validation
authorization/approval
Git versioning
Stronghold generation assignment
activation
runtime reconciliation
journaling
```

An API endpoint or GUI button does not become a privileged alternative path.

## Service Identities

Automation uses scoped service identities rather than shared human credentials.

A service identity must receive only the operations/targets required for its purpose.

Automation-originated durable changes require the same attributable change reason/comment semantics as human-originated changes. The reason may be supplied programmatically, but it remains mandatory and recorded.

## Break-Glass and Recovery

Break-glass/recovery capability is not an excuse for an undocumented unrestricted routine path.

Exceptional local-console or engineering operations remain explicitly governed, attributable, and journaled according to the recovery/support architecture.

Where an emergency path must bypass an unavailable external identity provider, Stronghold still records the local recovery identity, operation, target, reason, and result.

## Truth Separations

```text
candidate edited                     != configuration changed
validation passed                    != configuration saved
comment supplied                     != operation authorized
user authenticated                   != operation authorized
role/group membership                != unrestricted authority
edit permission                      != finalization permission
Git commit created                   != activation succeeded
configuration activated              != runtime reconciliation complete
Git history valid                    != administrative journal valid
saved configuration                  != operational intent proven correct
rollback                             != history deletion
object renamed                       != new object identity
historical configuration visible     != historical configuration restored
external DNP concepts reused         != Stronghold depends on DNP runtime
```

## Scope

This document freezes architecture only. It does not pull Git implementation, complete RBAC/authorization services, DNP code reuse, approval workflows, CLI/API/UI implementation, historical-query indexing, or post-commit analytics into Phase 0.
