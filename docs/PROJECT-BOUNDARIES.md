# Stronghold Component Boundaries

## Purpose

Stronghold is **one tightly coupled security platform** composed of multiple deployable and implementation components. Shared platform goals do not justify blurred implementation ownership, but implementation boundaries must not be mistaken for separate unrelated products.

The Stronghold platform currently contains three infrastructure components plus the endpoint Agent component:

```text
Stronghold FW
    physical observation / enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    third Stronghold infrastructure node
    supported customer VM or supported bare-metal server
    access-control / identity / authorization coordination
    synchronized endpoint-policy distribution / fleet convergence

Stronghold Agent
    endpoint component / endpoint PEP
    managed through Stronghold Access
    endpoint-applicable projection of finalized Stronghold policy
```

The components are intended to operate as one Stronghold platform through explicit authenticated/versioned contracts while preserving separate authority, failure domains, implementation trees, and qualification boundaries.

> **Tightly coupled platform does not mean monolithic software.**

## Distributed Policy Enforcement Boundary

Stronghold uses one governed policy/configuration authority with multiple enforcement projections.

A finalized Stronghold generation may contain policy applicable to Stronghold FW, one or more Stronghold Agents, network admission, secure access, or other later enforcement contexts. Stronghold Access coordinates distribution of the endpoint-applicable projection to managed Agents.

```text
FINALIZED STRONGHOLD GENERATION
        |
        +---- Stronghold FW projection
        |
        +---- device-specific Agent projection
```

The projections are not required to be byte-for-byte identical because the enforcement points have different responsibilities.

```text
same finalized policy authority
!=
identical enforcement representation

Agent policy projection
!=
complete FW configuration
```

Stronghold Agent may reject traffic locally when its active endpoint policy definitively denies the connection. In that case the packet need not be transmitted merely so Stronghold FW can deny it again.

```text
Agent DENY
    -> endpoint decision record
    -> packet not transmitted
    -> FW packet observation does not exist
```

A Stronghold operator view may surface the endpoint denial alongside FW activity, but it must not fabricate a FW denial or physical observation.

```text
endpoint DENY
!=
FW DENY

endpoint denial reported to platform
!=
packet presented to FW
```

A finalized generation should be pushed to connected Agents where appropriate. An Agent that misses the update because it is offline, asleep, or disconnected receives the required update during a later authenticated check-in when Stronghold Access detects generation drift.

```text
policy finalized
!=
every Agent synchronized

policy sent
!=
policy activated

policy activated
!=
enforcement healthy
```

The active endpoint projection continues to govern applicable local behavior when the managed endpoint leaves the organization's network, subject to explicit lease, holdover, expiration, and revocation semantics.

The authoritative cross-component contract is:

```text
docs/DISTRIBUTED-POLICY-ENFORCEMENT.md
```

## Stronghold Access Boundary

Stronghold Access is the third Stronghold infrastructure component when deployed.

It runs on a supported server or VM on the customer's network rather than inside the high-PPS Stronghold FW dataplane.

It owns access-control responsibilities including first-class 802.1X/network admission, AAA integration, identity inputs, device trust, posture, Access Sessions, resource authorization, revocation, Agent policy/session distribution, endpoint policy-generation synchronization, Agent check-in/convergence state, and the controlled FW integration contract.

Authoritative Access architecture:

```text
docs/access/ARCHITECTURE.md
```

Implementation boundary:

```text
go/access/
```

Stronghold Access is not embedded Stronghold FW code, but it is a Stronghold platform component rather than an unrelated standalone product.

## Stronghold Agent Boundary

Stronghold Agent is the endpoint-side Stronghold component and endpoint Policy Enforcement Point.

It is not the Stronghold Access server and is not Stronghold FW code.

It owns endpoint responsibilities including Windows-first endpoint identity/enrichment, process-aware endpoint enforcement, enforcement of the device-applicable projection of finalized Stronghold policy, off-network policy continuity, Protected Endpoint transport, endpoint health, and endpoint decision records.

Authoritative Agent architecture:

```text
docs/agent/ARCHITECTURE.md
```

Implementation boundary:

```text
go/agent/
```

The Agent is managed/coordinated through Stronghold Access and does not become an independent Stronghold Policy Engine.

## Stronghold FW Boundary

Stronghold FW is the live-network Stronghold appliance and network-side enforcement authority.

It owns physical-interface observation, AF_XDP packet acquisition, PCAP durability, routing, NAT, network firewall enforcement, near-real-time operational history, FW journals, and the network-side Policy Enforcement Point.

Access or Agent authorization does not bypass FW policy.

```text
Agent ALLOW
!=
FW ALLOW

Access GRANT
!=
FW ALLOW
```

Traffic already denied at the Agent does not need to be emitted merely to create a FW denial. Stronghold FW remains authoritative only for traffic actually presented to its supported network path.

## Stronghold Net-Hunter Boundary

Stronghold Net-Hunter is the historical Stronghold appliance.

It owns verified long-term packet/journal history, processing/reprocessing, correlation, hunting/query, historical reconstruction, and related derived analysis.

Net-Hunter does not become a synchronous requirement for FW packet acquisition, ordinary forwarding, Access policy evaluation, or Agent local enforcement.

Net-Hunter may correlate Agent decisions with FW decisions and packet history, but correlation must preserve the fact that an Agent-local DENY may correctly have no corresponding FW packet observation.

## Pathfinder Relationship

Iron Signal Systems Pathfinder is **not** a Stronghold platform component.

It is a separate ISS threat-intelligence system that may become a first-class intelligence/enrichment peer to Stronghold.

```text
Pathfinder
    threat-intelligence record / interpretation authority

Stronghold
    observation / access / enforcement / operational-history authority
```

The relationship is governed by:

```text
docs/PATHFINDER-INTEGRATION.md
```

Stronghold may consume Pathfinder intelligence for FW context, IDS/IPS enrichment, Net-Hunter retrospective matching, and Access risk/trust inputs. Stronghold may also submit qualified observables to Pathfinder.

Neither system rewrites the other's source truth.

## Legacy Documentation Precedence

Existing `docs/SECURE-ACCESS.md` and `docs/ENDPOINT-ENFORCEMENT.md` were written before Stronghold Access and Stronghold Agent were separated into explicit component ownership.

Where those older documents assign endpoint/access authority differently from the current platform architecture, the following documents govern:

```text
docs/PROJECT-BOUNDARIES.md
docs/DISTRIBUTED-POLICY-ENFORCEMENT.md
docs/access/ARCHITECTURE.md
docs/agent/ARCHITECTURE.md
docs/PATHFINDER-INTEGRATION.md
```

The older documents remain useful for previously defined secure-access, WireGuard, endpoint-enforcement, and FW integration concepts, but they must not be used to collapse Access, Agent, and FW into one implementation process or to describe them as unrelated commercial products.

## Cross-Component Contract Rule

Stronghold components are intended to cooperate tightly while remaining separate implementations.

```text
Stronghold Access
    <-> versioned authenticated policy/synchronization contract <-> Stronghold Agent

Stronghold Access
    <-> versioned authenticated authorization/context contract <-> Stronghold FW

Stronghold FW
    <-> verified history contract <-> Stronghold Net-Hunter

Stronghold / approved components
    <-> versioned authenticated intelligence contract <-> Pathfinder
```

Direct imports of another component's internal implementation packages are prohibited.

A future shared Stronghold schema/protocol package may exist only after the relevant cross-component contract is frozen, versioned, and explicitly owned.

## Governing Principle

> **Stronghold is one platform, not one process. FW, Net-Hunter, Access, and Agent remain explicit components with distinct responsibilities and authority while operating as one tightly integrated Stronghold system. A managed Agent enforces its endpoint-applicable projection of the finalized Stronghold policy wherever the endpoint is operating; Access keeps that projection synchronized; FW independently controls traffic actually presented to the network; and no component fabricates an action performed by another.**
