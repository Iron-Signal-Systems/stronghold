# Stronghold Access Go Project Rules

## Scope

This file governs implementation under:

```text
go/access/
```

This subtree is reserved for **Stronghold Access**, the standalone/bolt-on access-control system.

Stronghold Access is a separate project from:

```text
go/agent/
Stronghold FW implementation
Stronghold Net-Hunter implementation
```

Do not place Stronghold Agent endpoint implementation, Stronghold FW dataplane/capture code, or Net-Hunter processing code in this subtree.

## Product Responsibility

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
```

Stronghold Access does not own packet capture, routing, NAT, FW dataplane enforcement, endpoint WFP enforcement, or Hunter authoritative history.

See `docs/access/ARCHITECTURE.md`.

## Deployment Boundary

Stronghold Access is intended to support qualified deployment on either:

```text
customer VM
or
customer bare-metal host
```

The exact supported operating system, hypervisor matrix, storage/database model, HA model, and sizing profiles are not yet frozen.

Do not hard-code an unsupported deployment assumption into shared contracts before those decisions are made.

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

## Stronghold FW Boundary

Stronghold Access is a bolt-on peer to Stronghold FW, not part of the FW process image.

Access must not require unrestricted root, shell, package-manager, filesystem, or arbitrary-command authority over the FW.

The FW integration should use a narrowly scoped, authenticated control contract.

Preserve:

```text
Access GRANT != FW ALLOW
```

FW remains an independent network PEP.

## Shared-Code Rule

Do not create generic packages named `common`, `shared`, `util`, `helpers`, or `framework` merely to share implementation with Agent or FW.

If a real cross-project wire/schema contract becomes necessary:

1. freeze the contract;
2. define ownership and versioning;
3. keep implementation-specific behavior in its owning project;
4. avoid direct imports of another project's internal packages.

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
```

Do not create these packages before actual implementation work requires them.

## Truth Discipline

Stronghold Access must preserve what each source can actually establish.

Examples:

```text
AAA authenticated         != Stronghold authorized
certificate valid         != device authorized
posture unavailable       != posture passed
CoA requested             != CoA applied
policy sent               != policy activated
Agent reachable           != Agent enforcement healthy
FW reachable              != FW authorization granted
```

## Security Principle

> **Stronghold Access coordinates authorization. It does not erase or impersonate the independent authority of network admission, endpoint enforcement, or Stronghold FW.**
