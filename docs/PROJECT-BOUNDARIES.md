# Stronghold Product Boundaries

## Purpose

Stronghold is one platform with multiple cooperating products. Shared platform goals do not justify blurred implementation ownership.

Current product boundaries are:

```text
Stronghold FW
    physical enforcement appliance

Stronghold Net-Hunter
    historical preservation / processing / hunt appliance

Stronghold Access
    standalone or FW-integrated access-control system
    customer VM or supported bare-metal host

Stronghold Agent
    endpoint application / endpoint PEP
```

## Stronghold Access Boundary

Stronghold Access is not embedded Stronghold FW code.

It owns access-control responsibilities including first-class 802.1X/network admission, AAA integration, identity inputs, device trust, posture, Access Sessions, resource authorization, revocation, Agent policy/session distribution, and the controlled FW bolt-on interface.

Authoritative Access architecture:

```text
docs/access/ARCHITECTURE.md
```

Implementation boundary:

```text
go/access/
```

## Stronghold Agent Boundary

Stronghold Agent is not the Stronghold Access server and is not Stronghold FW code.

It owns endpoint responsibilities including Windows-first endpoint identity/enrichment, process-aware endpoint enforcement, Protected Endpoint transport, endpoint health, and endpoint decision records.

Authoritative Agent architecture:

```text
docs/agent/ARCHITECTURE.md
```

Implementation boundary:

```text
go/agent/
```

## Legacy Documentation Precedence

Existing `docs/SECURE-ACCESS.md` and `docs/ENDPOINT-ENFORCEMENT.md` were written before Stronghold Access and Stronghold Agent were separated into explicit product boundaries.

Where those older documents assign endpoint/access control authority differently from the newer product boundaries, the following documents govern the product split:

```text
docs/PROJECT-BOUNDARIES.md
docs/access/ARCHITECTURE.md
docs/agent/ARCHITECTURE.md
```

The older documents remain useful for previously defined secure-access, WireGuard, endpoint-enforcement, and FW integration concepts, but they must not be used to collapse Access, Agent, and FW back into one implementation project.

## Cross-Project Contract Rule

Products may cooperate deeply while remaining separate implementations.

```text
Stronghold Access
    <-> versioned authenticated contract <-> Stronghold Agent

Stronghold Access
    <-> versioned authenticated contract <-> Stronghold FW

Stronghold FW
    <-> verified history contract <-> Stronghold Net-Hunter
```

Direct imports of another product's internal implementation packages are prohibited.

A future shared schema/protocol package may exist only after the relevant cross-product contract is frozen, versioned, and explicitly owned.

## Governing Principle

> **One Stronghold platform does not mean one Stronghold program. Product responsibilities, authority, failure domains, release boundaries, and implementation trees remain explicit.**
