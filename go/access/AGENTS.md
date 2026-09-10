# Stronghold Access Go Component Rules

## Scope

This file governs implementation under:

```text
go/access/
```

This subtree is reserved for **Stronghold Access**, the Stronghold platform's access-control and authorization-coordination infrastructure component.

Stronghold Access is the third Stronghold infrastructure node when deployed and is intended to run on a supported customer VM or supported bare-metal server on the customer's network.

It is a separate implementation boundary from:

```text
go/agent/
Stronghold FW implementation
Stronghold Net-Hunter implementation
```

These boundaries do not make the components unrelated products. They exist to preserve ownership, authority, security, failure domains, and maintainability inside one tightly coupled Stronghold platform.

Do not place Stronghold Agent endpoint implementation, Stronghold FW dataplane/capture code, or Net-Hunter processing code in this subtree.

## Component Responsibility

Stronghold Access owns control-plane responsibilities such as:

```text
identity inputs
network admission / 802.1X
AAA integration
device trust
endpoint posture
Stronghold Access Sessions
resource authorization
policy evaluation
revocation / reevaluation
Agent policy/session distribution
controlled Stronghold FW integration
optional Pathfinder intelligence/risk inputs
```

Stronghold Access does not own packet capture, routing, NAT, FW dataplane enforcement, endpoint WFP enforcement, Hunter authoritative history, or Pathfinder threat-intelligence authority.

See:

```text
docs/PROJECT-BOUNDARIES.md
docs/access/ARCHITECTURE.md
docs/PATHFINDER-INTEGRATION.md
```

## Deployment Boundary

Stronghold Access is intended to support qualified deployment on either:

```text
customer VM
or
customer bare-metal server
```

The exact supported operating system, hypervisor matrix, storage/database model, HA model, and sizing profiles are not yet frozen.

Do not hard-code an unsupported deployment assumption into cross-component contracts before those decisions are made.

## First-Class 802.1X

802.1X/network admission is a first-class Access responsibility.

Do not reduce the model to a generic `radius server` configuration object.

Implementation must preserve distinct concepts such as:

```text
AAA Provider
Authenticator
Network Access Policy
Network Access Session
802.1X Identity
EAP Method
Certificate Identity
Authentication Result
Authorization Result
Network Assignment
CoA / Reauthentication
Session Termination
Admission Health
```

Preserve:

```text
802.1X authenticated != resource authorized
RADIUS Access-Accept != Stronghold Access GRANT
MAB admitted         != 802.1X authenticated
```

## AAA Boundary

External AAA is an input to Stronghold Access policy, not a bypass around it.

RADIUS/NPS/ISE/ClearPass/FreeRADIUS integration must preserve the source and meaning of returned facts/attributes.

TACACS+ administrative/device AAA must not be conflated with 802.1X network admission.

## Session Boundaries

Do not collapse:

```text
Network Access Session
Stronghold Access Session
Transport Session
```

into one generic session type merely because they can be correlated.

Each has different authority, lifecycle, and failure semantics.

## Stronghold Agent Boundary

Stronghold Agent implementation lives under:

```text
go/agent/
```

Stronghold Access code must not import Agent internal implementation packages.

Communication occurs through explicit versioned protocol/schema contracts.

Do not move endpoint WFP implementation into Access for convenience.

## Endpoint Policy Distribution

Stronghold Access owns policy evaluation and distribution to Stronghold Agent. It does not directly implement endpoint packet filtering.

Integrated policy flow should preserve:

```text
FW / network admission establishes network context
        ↓
Stronghold Access correlates context + identity + posture
        ↓
optional approved Pathfinder context contributes policy input
        ↓
Stronghold Access evaluates policy
        ↓
signed monotonic Agent policy generation
        ↓
Stronghold Agent verifies / validates / programs WFP
```

Do not use ad hoc per-packet commands from FW to Agent as the endpoint policy model.

Preserve:

```text
network context received  != endpoint policy evaluated
Pathfinder context received != Access decision made
policy generated          != policy delivered
policy delivered          != policy activated
policy activated          != Agent enforcement healthy
```

The endpoint must not be able to self-assert a trusted VLAN/zone and obtain broader policy. Network-side context must retain source, generation/freshness, and confidence sufficient for authorization use.

## Stronghold FW Boundary

Stronghold Access is a tightly integrated Stronghold control peer to FW, not part of the FW process image.

Access must not require unrestricted root, shell, package-manager, filesystem, or arbitrary-command authority over the FW.

The FW integration should use a narrowly scoped, authenticated control contract.

Preserve:

```text
Access GRANT  != FW ALLOW
endpoint ALLOW != FW ALLOW
```

FW remains an independent network PEP.

FW-supplied VLAN/zone/interface/address context may be an input to Access policy, but Access must preserve that context as a sourced fact rather than silently convert it into permanent endpoint trust.

## Pathfinder Boundary

Pathfinder is a separate Iron Signal Systems threat-intelligence authority, not a Stronghold component and not an Access Policy Engine.

Access may consume approved Pathfinder intelligence/risk context through the future versioned Stronghold↔Pathfinder contract.

Preserve:

```text
Pathfinder record exists           != observable malicious
Pathfinder malicious classification != compromise proven
Pathfinder risk signal             != Access Session revoked
Pathfinder unavailable             != observable trusted
```

If Pathfinder context influences a Stronghold Access decision, preserve sufficient provenance to identify the Pathfinder Record ID / interpretation generation used.

Do not implement direct Agent↔Pathfinder communication from this subtree merely for convenience. Agent receives the resulting authorized Stronghold policy through Access.

## Local Endpoint Enforcement Boundary

Access may authorize Agent-side process/application-aware connection enforcement so clearly unauthorized traffic can be stopped before reaching the LAN, WireGuard transport, or Stronghold FW.

That optimization does not move endpoint implementation into Access and does not change FW authority.

```text
endpoint DENY != FW observed DENY
endpoint ALLOW != FW ALLOW
```

Initial endpoint enforcement is not a requirement for general deep-payload inspection. A future local proxy or WFP callout-driver design requires separate approval and qualification.

## Shared-Code Rule

Do not create generic packages named `common`, `shared`, `util`, `helpers`, or `framework` merely to share implementation with Agent or FW.

If a real cross-component wire/schema contract becomes necessary:

1. freeze the contract;
2. define ownership and versioning;
3. keep implementation-specific behavior in its owning component;
4. avoid direct imports of another component's internal packages.

## Go Engineering

All parent `go/AGENTS.md` requirements apply unless this file narrows them for Stronghold Access.

Prefer focused packages representing real Access responsibilities.

Potential future package directions, only when roadmap work requires them, may include:

```text
internal/admission/
internal/aaa/
internal/authenticator/
internal/identity/
internal/policy/
internal/posture/
internal/revocation/
internal/session/
internal/transport/
internal/intelligence/
```

Do not create these packages before actual implementation work requires them.

## Truth Discipline

Stronghold Access must preserve what each source can actually establish.

Examples:

```text
AAA authenticated          != Stronghold authorized
certificate valid          != device authorized
posture unavailable        != posture passed
CoA requested              != CoA applied
FW reports VLAN/zone       != endpoint permanently trusted
Pathfinder match           != endpoint compromised
Pathfinder risk signal     != Access REVOKE performed
policy sent                != policy activated
Agent reachable            != Agent enforcement healthy
FW reachable               != FW authorization granted
endpoint process allowed   != FW traffic allowed
```

## Security Principle

> **Stronghold Access coordinates authorization across the Stronghold platform. It does not erase or impersonate the independent authority of network admission, endpoint enforcement, Stronghold FW, Net-Hunter history, or Pathfinder threat-intelligence interpretation.**
