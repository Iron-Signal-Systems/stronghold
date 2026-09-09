# Stronghold Policy Simulation and Dry Run Architecture

## Purpose

This document defines the architectural contract for Stronghold FW policy simulation, candidate impact analysis, counterfactual testing, and pre-activation Dry Run behavior.

Implementation remains deferred from Phase 0. This document freezes the semantics that later CLI, API, UI, policy, routing, NAT, state, DNS/FQDN, secure-access, and post-commit analysis implementations must preserve.

The governing rules are:

> **Dry Run evaluates the candidate configuration as though it were authoritative, but it does not make that candidate authoritative.**

> **Simulation predicts expected behavior; it does not claim that predicted behavior actually occurred.**

> **Dry Run must not mutate production dataplane, configuration, session, counter, route, NAT, tunnel, HA, or packet-history state.**

> **Activation and post-commit observation remain the authoritative proof of what Stronghold actually did.**

## Management Workflow

The intended Stronghold change workflow is:

```text
EDIT CANDIDATE
      ↓
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
GIT-BACKED FINALIZATION
      ↓
NEW STRONGHOLD GENERATION
      ↓
ACTIVATE
      ↓
RUNTIME RECONCILIATION
      ↓
POST-COMMIT OBSERVATION
      ↓
COMPARE EXPECTED vs ACTUAL
```

Simulation is available before durable finalization so an administrator can inspect likely consequences while the candidate is still safe to edit or discard.

A simulation is not a durable configuration change and therefore does not require a configuration-change comment merely to run it. Simulation access may still require scoped read/simulation authority because the output can expose sensitive network, policy, route, DNS, and session information.

```text
simulation authorized
!=
configuration finalization authorized
```

## Single-Flow Policy Test

Stronghold should support testing a hypothetical flow against running or candidate configuration.

Conceptual CLI example:

```text
test policy
 source HR-PC-17
 destination payroll.vendor.com
 service HTTPS
```

The exact command syntax is not frozen.

A useful result explains the decision path rather than returning only `ALLOW` or `DENY`:

```text
Candidate Base Generation: 842
Candidate Identity:         pending / candidate hash

Source:
    HR-PC-17
    10.20.30.17
    Zone: HR

Destination:
    payroll.vendor.com

Policy evaluation:
    Policy 10     NO_MATCH
    Policy 20     NO_MATCH
    Policy 37     ALLOW

Route:
    0.0.0.0/0
    next-hop 203.0.113.1

WAN:
    WAN1

NAT:
    NAT-04
    expected SNAT 203.0.113.20

EXPECTED RESULT:
    ALLOW
```

Where required, Stronghold should also explain why competing policies, routes, WANs, NAT rules, or other candidate treatments were not selected.

## Address-Family-Aware FQDN Simulation

FQDN policy simulation must be address-family aware.

An FQDN may resolve to independent IPv4 and IPv6 answers:

```text
payroll.vendor.com

A:
    192.0.2.50

AAAA:
    2001:db8:500::50
```

Stronghold must not collapse this into a generic `FQDN resolved` state.

Where both families are returned, Dry Run should evaluate both families independently when applicable:

```text
IPv4 evaluation:
    Policy 37     ALLOW
    Route         AVAILABLE
    WAN           WAN1
    NAT           NAT-04
    EXPECTED      ALLOW

IPv6 evaluation:
    Policy 37     ALLOW
    Route         NONE
    EXPECTED      NO_IPV6_ROUTE
```

Stronghold should surface actionable mismatches such as:

```text
WARNING

payroll.vendor.com is dual-stack.

IPv4 expected result: ALLOW
IPv6 expected result: NO_IPV6_ROUTE

Dual-stack clients may attempt IPv6.
```

The same principle applies when policy coverage, route availability, WAN eligibility, tunnel requirements, source validation, or other behavior differs by family.

Mandatory separations include:

```text
FQDN resolved                 != IPv4 resolved
FQDN resolved                 != IPv6 resolved
A record authorized           != AAAA record authorized
IPv4 route available          != IPv6 route available
IPv4 application success      != IPv6 application success
DNS answer available          != usable network path
hostname looks valid          != both address families are usable
```

The exact DNS/FQDN runtime contract remains separate future architecture work. Simulation must consume and report the DNS inputs it actually used rather than inventing a timeless hostname-to-address truth.

## Candidate Against Current Sessions

Stronghold should support evaluating the candidate configuration against current live-session facts without modifying those sessions.

Conceptual operation:

```text
simulate candidate against current sessions
```

Potential summary:

```text
Candidate Impact Estimate

Base Generation:       842
Candidate Changes:       7

Current Sessions:    8,421
Sessions Evaluated:  8,421

Unaffected:          7,982
Still Allowed:         407
Would Rebind:          366
Would Restart:          29
Would Terminate:        12

Reasons:
    Explicit deny:              4
    Default deny:               8
    NAT treatment changed:     17
    WAN binding changed:        9
    inspection/tunnel change:   3
```

The categories must align with `docs/STATEFUL-ENFORCEMENT.md`:

```text
REBOUND
RESTART_REQUIRED
TERMINATE
```

A current session's existence is an input to simulation, not authority to keep it alive.

## Session Drill-Down

Administrators should be able to inspect the sessions expected to change outcome.

Conceptual result:

```text
SESSION        SOURCE       DESTINATION       SERVICE   CURRENT    CANDIDATE
S-88291        HR-PC-17     payroll.vendor    HTTPS     P-120      DEFAULT_DENY
S-88302        HR-PC-22     payroll.vendor    HTTPS     P-120      DEFAULT_DENY
S-88910        FIN-PC-04    ERP-SERVER        HTTPS     P-201      P-199 DENY
```

Where the candidate remains `ALLOW` but established dataplane treatment cannot be reused, Stronghold should identify the reason for `RESTART_REQUIRED` rather than mislabeling the session as denied.

## Rule Move Simulation

Rule position is enforcement behavior, not cosmetic metadata.

If a candidate moves a rule, Dry Run should identify policies and sessions whose first-match result changes.

Example:

```text
Policy P-199

Current position:    220
Candidate position:   90

Policies affected/shadowed by this move:
    P-120
    P-134

Existing sessions whose first-match result changes:
    91

Would remain allowed:
    74

Would terminate:
    17

New candidate result for terminated set:
    Explicit DENY
```

The exact shadow/overlap terminology remains implementation work, but Stronghold should explain the changed evaluation order.

## Rule Removal Simulation

Deleting a policy should expose expected runtime cleanup before the administrator activates the candidate.

Example:

```text
Policy P-120 will be removed.

Current sessions bound to P-120:
    145

Candidate reconciliation:
    Rebind to P-181:       113
    Rebind to P-190:        21
    Default deny:            8
    Restart required:        3

Expected terminated sessions:
    11
```

This is more useful than a generic confirmation prompt because it describes likely consequences rather than merely asking whether the operator intended to click `delete`.

## Referenced Object Simulation

Changes to shared objects may alter many policies indirectly.

Examples include:

```text
host/network objects
service objects
zones
interfaces
VLANs
FQDN objects
route objects
WAN preference objects
NAT objects
secure-access resources
future inspection profiles
```

If `HR-NETWORK` changes from:

```text
10.20.30.0/24
```

to:

```text
10.20.30.0/25
```

Stronghold should be able to report candidate dependency and impact information such as:

```text
Object HR-NETWORK changed.

Referenced by:
    14 security policies
     2 NAT policies
     1 secure-access resource definition

Current sessions affected:
    218

Sessions whose source no longer belongs to HR-NETWORK:
    37

Candidate outcomes:
    Rebound:       9
    Terminated:   28
```

Dependency discovery and impact simulation are related but distinct:

```text
object referenced
!=
object change alters current traffic outcome
```

## Route Simulation

Candidate route changes should be testable without programming the production FIB.

Simulation may evaluate:

```text
selected prefix
route source
next hop
outbound interface
health input where modeled
metric/tie-break
WAN relationship
existing session route binding
```

Potential output:

```text
Candidate route changes affect:
    318 current flows

Expected WAN1 -> WAN2 selection for new flows:
    318

Existing sessions:
    preserve binding:      297
    restart required:       21
```

The simulator must not imply that a route grants authorization.

```text
route selected
!=
traffic authorized
```

## NAT Simulation

Candidate NAT behavior must be simulated separately from policy authorization.

Example:

```text
NAT-04 changed:
    translated source
    203.0.113.20
        ->
    203.0.113.21

Existing sessions using NAT-04:
    146

Would require restart:
    146
```

A candidate can remain policy-allowed while requiring established sessions to restart because NAT identity or mapping cannot safely change mid-session.

```text
candidate remains ALLOW
!=
existing NAT state remains reusable
```

## WAN and Path Simulation

Stronghold should be able to explain expected WAN/path selection under the candidate configuration while preserving:

```text
path available
!=
path authorized
```

Simulation may evaluate destination/service eligibility, preference, route availability, health inputs, and existing-session binding.

Dynamic health can change after simulation, so the result remains predictive rather than authoritative.

## Secure-Access Simulation

Future zero-trust remote-access simulation should evaluate the same separation preserved in `docs/SECURE-ACCESS.md`:

```text
WireGuard transport available
!=
user authorized

device authorized
!=
resource authorized

resource authorized
!=
application connection succeeded
```

A future simulation may evaluate a subject/device/resource/service request against candidate policy without creating an Access Session, programming WireGuard state, or authorizing production traffic.

## Simulation Must Be Non-Mutating

Dry Run must not intentionally modify production behavior or production usage statistics.

At minimum it must not:

```text
terminate sessions
create production sessions
modify conntrack/session state
change NAT mappings
program production routes/FIB
change WAN selection for live traffic
program nftables enforcement state
modify WireGuard production peers/sessions
modify site-tunnel state
change HA ownership/state
activate DNS-derived policy objects
increment production policy hit counters
update production last-match timestamps
create production authorization leases/sessions
alter authoritative packet history
```

Simulation may use isolated ephemeral computation/state required to produce the result.

Any simulation-specific counters or records must remain distinguishable from production runtime counters.

```text
production hit
!=
simulation match
```

## Simulation Authorization and Journaling

Simulation itself may reveal sensitive network information and therefore remains an authorized management operation.

Stronghold should preserve attributable simulation records sufficient to establish, as appropriate:

```text
Simulation ID
actor/service identity
management surface
candidate base generation
candidate identity/hash
simulation class
request context
start/end time
result status
```

The simulation record belongs to administrative/operational history according to the later journal schema.

Running a simulation does not create a configuration generation.

```text
simulation performed
!=
configuration changed
```

A mandatory configuration-change comment is not required merely to run a non-mutating simulation. If the operator later finalizes the candidate, the normal mandatory comment and authorization contract applies independently.

## Prediction Vocabulary

Stronghold must label simulation results as predictions.

Preferred conceptual vocabulary includes:

```text
EXPECTED RESULT
ESTIMATED IMPACT
WOULD ALLOW
WOULD DENY
WOULD REBIND
WOULD RESTART
WOULD TERMINATE
```

Stronghold should not label a simulated result as though a packet actually traversed the appliance.

Conditions may change between simulation and activation, including:

```text
DNS answers / TTL
route state
dynamic routing
WAN health
interface state
session population
HA state
time/trust state
identity/posture state
external dependencies
```

Therefore:

```text
simulation predicts ALLOW
!=
packet was actually allowed

candidate impact estimate
!=
post-commit observed impact
```

## Expected Versus Actual Comparison

The strongest operational workflow is to compare pre-commit expectation with post-commit observation.

Conceptually:

```text
BEFORE
    candidate + live facts
        ↓
    simulation
        ↓
    expected impact

CHANGE
    finalize + authorize + activate

AFTER
    runtime reconciliation
        ↓
    near-real-time traffic observation
        ↓
    actual allow/deny/result history
        ↓
    compare expected vs actual
```

Potential result:

```text
Generation 843

SIMULATED:
    12 sessions expected terminated

ACTUAL:
    12 sessions terminated

SIMULATED:
    payroll.vendor.com IPv6 had no route

ACTUAL:
    4 IPv6 attempts observed
    4 failed NO_IPV6_ROUTE
```

More importantly, Stronghold should surface material divergence:

```text
SIMULATION EXPECTED:
    0 new denies

ACTUAL SINCE COMMIT:
    38 newly denied flows

ATTENTION:
    observed impact differs from simulation
```

This warning does not automatically declare the change wrong. It tells the administrator that the observed network did not behave as predicted and should be reviewed while the change context is still fresh.

## Packet-History Correlation

Post-activation validation may correlate the candidate simulation with authoritative Stronghold packet history and decision journals.

Simulation output itself is not packet history.

```text
simulation input
    ↓
expected decision

actual physical observation
    ↓
actual Stronghold decision
    ↓
actual forwarding/result
```

An administrator should eventually be able to pivot from a simulation expectation to the post-commit flows that matched, diverged, failed, or were not observed.

## Stale Simulation Detection

A simulation is bound to the candidate and relevant base/runtime context used to produce it.

If another administrator changes the running generation, the candidate changes, or material dependent state changes before finalization, Stronghold should mark the prior simulation stale where it can establish that fact.

```text
simulation valid for candidate hash A
!=
simulation valid for candidate hash B

simulation against generation 842
!=
simulation against generation 843
```

Stronghold may require or strongly recommend re-simulation after material candidate/base changes according to later product policy.

A prior successful simulation must never become an authorization token permitting a stale candidate to activate.

## Resource Protection

Simulation can be computationally expensive, especially when evaluating large session populations, complex object dependencies, DNS/FQDN families, or route/NAT state.

Simulation work is secondary to Stronghold capture and live enforcement.

It must be bounded, observable, cancellable where appropriate, and must not starve:

```text
packet acquisition
active PCAP writes
live security enforcement
essential journal durability
HA heartbeat/control
critical management recovery paths
```

A simulation that cannot complete within available resource limits should return a truthful incomplete/degraded result rather than pretending complete analysis.

Potential states include:

```text
COMPLETE
PARTIAL
STALE
CANCELLED
RESOURCE_LIMITED
DEPENDENCY_UNAVAILABLE
```

Exact vocabulary remains implementation work.

## HA

Simulation is logically a management/control-plane operation and must not independently alter HA state.

Where current session/state data from an active/standby pair is used, Stronghold must identify the source and freshness of the state being simulated.

A simulation on stale standby state does not prove active-node impact.

## Management-Surface Consistency

CLI, API, and future UI expose the same underlying simulation authority and semantics.

No management surface may provide a hidden `test` operation that actually mutates production state while another surface is read-only.

```text
CLI Dry Run
API Dry Run
UI Dry Run
```

must be clients of the same Stronghold simulation engine/contract, subject to the same authorization and truth boundaries.

## Truth Separations

```text
simulation performed                    != configuration changed
simulation authorized                   != activation authorized
simulation match                        != production policy hit
simulation predicts ALLOW               != packet actually allowed
simulation predicts DENY                != packet actually denied
candidate impact estimate               != post-commit observed impact
FQDN resolved                           != both address families usable
A record authorized                     != AAAA record authorized
route selected                          != traffic authorized
candidate remains ALLOW                 != old dataplane state reusable
object referenced                       != object change alters traffic outcome
simulation complete                     != future runtime conditions unchanged
simulation successful                   != candidate authorized for activation
simulation against old base generation  != simulation current after base change
```

## Scope

This document freezes architecture only. It does not pull a policy simulator, session-impact engine, route/NAT simulator, DNS/FQDN engine, secure-access simulator, expected-vs-actual analytics, CLI/API/UI implementation, or simulation journal schema into Phase 0.
