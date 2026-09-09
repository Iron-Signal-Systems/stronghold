# Stronghold Stateful Enforcement and Policy Reconciliation

## Purpose

This document defines the architectural contract for Stronghold FW stateful enforcement and the behavior of live traffic when policy changes.

Implementation remains deferred from Phase 0. The purpose of this document is to freeze the authority and reconciliation semantics that later firewall/state implementation must preserve.

The governing rule is:

> **Existing state does not outrank current authorization.**

A session that was valid under an earlier policy generation does not retain permission merely because it already exists.

`docs/POLICY-SIMULATION.md` governs non-mutating pre-commit simulation of these same reconciliation outcomes and expected-versus-actual comparison after activation.

## Current Policy Generation Is Authoritative

Stronghold security policy is explicit-allow/default-deny and evaluated lowest-position-first, first match wins.

Each active session must retain enough policy lineage to establish at minimum:

```text
Session ID
Policy ID that authorized the session
Policy Generation that authorized the session
policy action
source / destination / service facts
route / WAN binding where applicable
NAT binding where applicable
relevant inspection / tunnel treatment where applicable
creation time
last activity time
current authorization/reconciliation state
```

The stored policy binding explains how the session was originally authorized. It does not create permanent authority.

Mandatory separation:

```text
session established under old policy
!=
session currently authorized
```

## Policy-Change Reconciliation

A committed policy change that may alter the outcome of existing traffic causes affected live state to be reconciled against the new authoritative policy generation.

Reconciliation may be required by changes including:

```text
rule creation
rule modification
rule movement / priority change
rule enable / disable
rule removal
referenced host/network/group change
referenced service/group change
zone/interface relationship change
FQDN-derived policy input change where policy semantics require it
NAT behavior change
WAN/path treatment change
secure-access authorization change
future inspection requirement change
```

The exact affected-session selection algorithm remains implementation work, but Stronghold must not knowingly preserve stale authorization merely to avoid reconciliation cost.

## Commit and Runtime Sequence

Conceptually:

```text
CANDIDATE
    ↓
VALIDATE
    ↓
SHOW DIFF
    ↓
DRY RUN / EXPECTED IMPACT
    ↓
AUTHORIZED SAVE / COMMIT
    ↓
NEW POLICY GENERATION BECOMES AUTHORITATIVE
    ↓
new traffic evaluates against new generation immediately
    ↓
identify affected existing sessions
    ↓
re-evaluate against new generation
    ↓
REBIND / RESTART_REQUIRED / TERMINATE
    ↓
RECONCILIATION COMPLETE
    ↓
POST-COMMIT ACTUAL IMPACT
```

New traffic must not continue using the prior generation while existing-session reconciliation is running.

Dry Run does not create the authoritative generation and does not perform real reconciliation. It predicts the outcomes defined in this document using the candidate and available current facts.

## Reconciliation Outcomes

### REBOUND

If the current policy still allows the session and the required dataplane treatment is safe to preserve, the session may continue under the newly matching Policy ID/generation.

Examples of state that may be safe to update in place when qualification proves it:

```text
Policy ID / generation attribution
rule position / effective priority
logging/accounting attribution
QoS or traffic-class metadata where safe
journal attribution
selected non-disruptive policy metadata
```

The rebinding event is attributable and observable.

### TERMINATE

The session is terminated when the current policy result is:

```text
explicit DENY
no matching ALLOW / default deny
resource authorization revoked
required security condition no longer satisfied
```

Termination is deliberate Stronghold enforcement, not an incidental conntrack timeout.

### RESTART_REQUIRED

The current policy may still allow the traffic, but the new required dataplane treatment may be incompatible with preserving the established session.

Examples may include:

```text
NAT identity/mapping changes
WAN/session binding changes
secure tunnel requirement changes
proxy/inspection path changes
routing-domain changes
other stateful treatment that cannot safely change mid-session
```

In this case Stronghold terminates the existing session and permits the application to establish a new session under the current generation.

```text
new policy allows
!=
old dataplane state is reusable
```

These same outcome classes are used by Dry Run as predictive categories:

```text
WOULD REBIND
WOULD RESTART
WOULD TERMINATE
```

A predicted outcome is not recorded as though the actual runtime event occurred.

## Rule Move and Rule Removal Semantics

Rule movement is an enforcement change, not a cosmetic configuration operation.

Example:

```text
10  USERS -> WEB       ALLOW
20  USERS -> ANY       ALLOW
30  USERS -> DATABASE  DENY
```

If the deny rule is moved to position 15:

```text
10  USERS -> WEB       ALLOW
15  USERS -> DATABASE  DENY
20  USERS -> ANY       ALLOW
```

existing database sessions previously authorized by Policy 20 are re-evaluated. They now match the deny and are terminated.

If a rule is removed and another current ALLOW rule legitimately covers the session, Stronghold may rebind the session to that rule when dataplane semantics remain safe.

If no current rule permits the traffic, Stronghold terminates it under default deny.

Therefore:

```text
rule removed
!=
session automatically dropped without reevaluation

rule removed
!=
old session remains authorized
```

The outcome is determined by the current policy generation.

Before finalization, Dry Run should be able to estimate the same consequences against current session facts so the operator can see likely rebinding, restart, and termination counts before activation.

## Runtime Policy Lifecycle

Configuration state and runtime cleanup state are separate.

Conceptual configuration state may include:

```text
CONFIGURED
DISABLED
REMOVED
```

Conceptual runtime state may include:

```text
ACTIVE
RECONCILING
TERMINATING
INACTIVE
```

A removed policy may therefore temporarily report:

```text
Policy ID:          P-01872
Config State:       REMOVED
Runtime State:      RECONCILING
Sessions Remaining: 17
```

and later:

```text
Policy ID:          P-01872
Config State:       REMOVED
Runtime State:      INACTIVE
Sessions Remaining: 0
```

Historical policy identity remains queryable after removal.

Simulation state is not a runtime lifecycle state for the production policy. A candidate policy that is being simulated does not become `ACTIVE` merely because a Dry Run completed.

## Policy and Session Counters

Stronghold should expose enough policy runtime information to establish whether a policy change has fully taken effect.

Potential counters/state include:

```text
Policy ID
current position where configured
current configuration state
current runtime state
current session count
current flow count
sessions created
sessions rebound in
sessions rebound out
sessions terminated by policy reconciliation
last matched time
last session created time
last session terminated time
reconciliation generation
reconciliation start/end time
sessions remaining to reconcile
```

Exact counter names and storage model remain implementation work.

The operator must be able to distinguish a policy that is merely absent from the candidate/running text from a policy whose live runtime state has actually been cleaned up.

Simulation matches/counters must remain separate from production counters.

```text
simulation match
!=
production policy hit
```

## Termination and Reconciliation Reasons

Stronghold should preserve explicit reasons rather than collapse all session loss into timeout/reset.

Potential reasons include:

```text
POLICY_REMOVED
POLICY_DISABLED
POLICY_REORDERED
POLICY_CHANGED_NO_LONGER_AUTHORIZED
POLICY_RECONCILIATION_DEFAULT_DENY
POLICY_RECONCILIATION_EXPLICIT_DENY
RESOURCE_AUTHORIZATION_REVOKED
DATAPLANE_RESTART_REQUIRED
ADMINISTRATIVE_TERMINATION
```

Exact vocabulary remains to be frozen with the implementation schema.

Simulation may predict these reason classes, but prediction must remain clearly distinguishable from an actual termination/reconciliation record.

## Pre-Commit Candidate Impact

Stronghold should allow a candidate to be evaluated against current sessions before finalization without mutating those sessions.

Potential output includes:

```text
Candidate Impact Estimate

Sessions Evaluated:  8,421
Unaffected:          7,982
Would Rebind:          366
Would Restart:          29
Would Terminate:        12
```

The simulator may also explain rule-order changes, default-deny results, NAT/WAN changes, route effects, object dependency effects, and FQDN/IPv4/IPv6 family differences according to `docs/POLICY-SIMULATION.md`.

A simulation is bound to the candidate/base/runtime context used to produce it and can become stale before activation.

```text
candidate impact estimate
!=
future runtime guaranteed
```

## Post-Commit Impact

Stronghold should make the operational consequences of a configuration generation visible while the administrator still has context for the change.

Potential post-commit information includes:

```text
changed Policy IDs
sessions evaluated
sessions rebound
sessions requiring restart
sessions terminated
newly allowed flows
newly denied flows
top affected sources/destinations/services
first affected observation time
reconciliation status
expected-versus-actual divergence
```

This is observation/reporting, not an automatic declaration that the administrator's change was good or bad.

A configuration commit may be syntactically valid and successfully activated while still producing an unintended operational result.

```text
configuration accepted
!=
operational intent proven correct
```

Where a pre-commit Dry Run exists, Stronghold should be able to compare the predicted reconciliation/traffic effects with actual observed behavior after activation.

Example:

```text
SIMULATION EXPECTED:
    0 new denies

ACTUAL SINCE COMMIT:
    38 newly denied flows

ATTENTION:
    observed impact differs from simulation
```

This warning does not itself declare the change wrong or trigger rollback.

## Packet-History Correlation

Stronghold's capture-first architecture should allow a policy change to be correlated with near-real-time observed traffic and decision history.

Conceptually:

```text
PRE-COMMIT SIMULATION
        ↓
EXPECTED IMPACT
        ↓
CONFIGURATION GENERATION CHANGE
        ↓
RUNTIME RECONCILIATION
        ↓
NEW ALLOW / DENY DECISIONS
        ↓
SOURCE / DESTINATION / SERVICE
        ↓
AUTHORITATIVE PACKET HISTORY
        ↓
EXPECTED vs ACTUAL
```

An operator investigating a change should be able to determine, subject to capture/history availability:

```text
what configuration changed
what Stronghold predicted before activation
which Policy IDs changed
which sessions were affected
which new flows were allowed/denied afterward
what packets were actually presented
what Stronghold decided
what Stronghold actually did
where observed behavior differed from prediction
```

This enables prompt discovery of mistaken rule movement/removal instead of waiting for long-lived client sessions to close hours later.

Simulation output is not authoritative packet history or an actual traffic decision.

## Rollback and Reconciliation

Rollback never rewrites history.

Restoring earlier configuration content creates a new configuration/Git state and a new Stronghold generation.

Example:

```text
Generation 841    known-good content
Generation 842    mistaken policy change
Generation 843    restore content based on 841
```

Generation 842 remains part of history.

Generation 843 triggers the same current-generation reconciliation contract as any other commit and may itself be Dry Run simulated before finalization.

## HA

Active/standby HA must eventually preserve the distinction between:

```text
session authorization state
session dataplane state
policy generation
reconciliation progress
state-sync progress
```

A standby must not silently preserve sessions under authorization that the active/current policy generation has revoked.

Exact synchronization, failover-during-reconciliation, and termination propagation behavior remain later qualification work.

Simulation that consumes HA state must identify the state source/freshness and must not claim active-node impact from stale standby data.

## Resource Protection

Policy reconciliation must be bounded and must not undermine Stronghold's capture-first resource priorities.

A large policy change may create substantial reconciliation work. Stronghold must expose backlog/progress truthfully rather than silently starving capture or claiming cleanup complete before it is complete.

If reconciliation cannot keep pace, Stronghold must expose a degraded/lagging state and continue to apply the current policy generation to new traffic.

Dry Run can also be computationally expensive. Simulation is secondary work and must not starve packet acquisition, durable PCAP writes, live enforcement, essential journal durability, HA heartbeat/control, or critical management recovery paths.

## Truth Separations

```text
session exists                         != session remains authorized
old policy permitted                   != current policy permits
rule removed                           != runtime state cleaned up
rule moved                             != cosmetic change
new allow match                        != old dataplane state reusable
simulation predicts REBOUND            != session actually rebound
simulation predicts TERMINATE          != session actually terminated
simulation match                       != production policy hit
candidate impact estimate              != post-commit observed impact
policy generation activated            != reconciliation complete
configuration accepted                 != operational intent proven correct
session terminated                     != packet history lost
rollback performed                     != mistaken generation erased
current session count zero             != policy never matched historically
```

## Scope

This document freezes architecture only. It does not pull the state engine, conntrack implementation, session schema, HA state synchronization, policy simulator, policy-impact analytics, or post-commit UI into Phase 0.
