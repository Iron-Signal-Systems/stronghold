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

Stronghold Agent
    endpoint component / endpoint PEP
    managed through Stronghold Access
```

The components are intended to operate as one Stronghold platform through explicit authenticated/versioned contracts while preserving separate authority, failure domains, implementation trees, and qualification boundaries.

> **Tightly coupled platform does not mean monolithic software.**

## Stronghold Access Boundary

Stronghold Access is the third Stronghold infrastructure component when deployed.

It runs on a supported server or VM on the customer's network rather than inside the high-PPS Stronghold FW dataplane.

It owns access-control responsibilities including first-class 802.1X/network admission, AAA integration, identity inputs, device trust, posture, Access Sessions, resource authorization, revocation, Agent policy/session distribution, and the controlled FW integration contract.

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

It owns endpoint responsibilities including Windows-first endpoint identity/enrichment, process-aware endpoint enforcement, Protected Endpoint transport, endpoint health, and endpoint decision records.

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

## Stronghold Net-Hunter Boundary

Stronghold Net-Hunter is the historical Stronghold appliance.

It owns verified long-term packet/journal history, processing/reprocessing, correlation, hunting/query, historical reconstruction, and related derived analysis.

Net-Hunter does not become a synchronous requirement for FW packet acquisition, ordinary forwarding, Access policy evaluation, or Agent local enforcement.

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
docs/access/ARCHITECTURE.md
docs/agent/ARCHITECTURE.md
docs/PATHFINDER-INTEGRATION.md
```

The older documents remain useful for previously defined secure-access, WireGuard, endpoint-enforcement, and FW integration concepts, but they must not be used to collapse Access, Agent, and FW into one implementation process or to describe them as unrelated commercial products.

## Cross-Component Contract Rule

Stronghold components are intended to cooperate tightly while remaining separate implementations.

```text
Stronghold Access
    <-> versioned authenticated contract <-> Stronghold Agent

Stronghold Access
    <-> versioned authenticated contract <-> Stronghold FW

Stronghold FW
    <-> verified history contract <-> Stronghold Net-Hunter

Stronghold / approved components
    <-> versioned authenticated intelligence contract <-> Pathfinder
```

Direct imports of another component's internal implementation packages are prohibited.

A future shared Stronghold schema/protocol package may exist only after the relevant cross-component contract is frozen, versioned, and explicitly owned.

## Governing Principle

> **Stronghold is one platform, not one process. FW, Net-Hunter, Access, and Agent remain explicit components with distinct responsibilities and authority while operating as one tightly integrated Stronghold system.**
