# Stronghold Architecture

## Purpose

This document records the current high-level architecture for the complete Stronghold system while preserving the capture-first engineering sequence already established for implementation.

Stronghold is best defined as:

> **An enforcement system that preserves the traffic presented to it before deciding what that traffic means or what should happen to it.**

Stronghold is a two-appliance network security system:

- **Stronghold FW** protects and observes the live network; and
- **Stronghold Net-Hunter** receives, verifies, preserves, processes, reprocesses, and exposes historical network activity for hunt/query use.

The complete product is organized around five responsibilities:

```text
OBSERVE
   ↓
RECORD
   ↓
ENFORCE
   ↓
REMEMBER
   ↓
HUNT
```

A concise practitioner expression of the product philosophy is:

> **Observe the truth. Preserve the history. Explain the decision.**

## Stronghold Truth Model

Stronghold deliberately separates three categories of truth:

```text
WHAT WAS PRESENTED
    authoritative physical-interface observation / PCAPNG

WHAT STRONGHOLD DID
    authoritative operational and decision journals

WHAT STRONGHOLD UNDERSTOOD
    derived interpretation, correlation, enrichment, and detection
```

### Observation survives interpretation

> **Observation survives interpretation.**

If Stronghold preserves a frame that it cannot decode today, that packet remains authoritative history. A later decoder may produce a better derived interpretation without rewriting the original observation.

```text
ORIGINAL PCAP
      ↓
NEW DECODER / CORRELATOR
      ↓
NEW DERIVED INTERPRETATION
```

The original observation does not change merely because Stronghold's understanding improves.

### Missing information remains explicit

Stronghold must not turn absence into inference.

Examples include:

```text
OBSERVED
DECODER_NOT_AVAILABLE
```

```text
OBSERVED
POLICY_DECISION_RECORD_INCOMPLETE
CAPTURE_REMAINS_AVAILABLE
```

```text
CAPTURE_GAP
reason = KERNEL_DROP
```

```text
DURABLE_CAPTURE = FAILED
reason = STORAGE_EXHAUSTED
```

```text
TIME_CONFIDENCE = DEGRADED
```

> **Missing interpretation must never erase observation, and missing history must never be presented as proof of absence.**

## Governing Principles

### Capture first

> **Capture first. Never sacrifice observation for secondary work.**

Stronghold prioritizes packet acquisition and durable capture over transfer, compression, indexing, analytics, hunt activity, and other background work. Net-Hunter must never become a runtime dependency for forwarding or capture on Stronghold FW.

### Authorization before normal forwarding work

> **A packet does not earn routing, NAT, or deeper forwarding work merely because it is technically routable. Stronghold observes it first, then requires policy authorization before normal forwarding work proceeds.**

> **Denied traffic should be cheap to reject, but never invisible.**

Observation is before interpretation; authorization is before unnecessary forwarding/deeper work. Stronghold may establish only the facts required to reach the earliest valid policy decision without making a successful route/FIB lookup itself constitute authorization.

### Explicit path permission

> **Path availability is not path permission.**

A WAN, route, interface, or alternate path is not eligible merely because Stronghold can technically reach it. Administrative policy determines which paths are permitted for a destination/service.

### Journal, do not casually log

> **Stronghold operational history is journaled, not treated as an overwriteable catch-all log.**

Administrative actions, system/health state, traffic decisions, trust/identity changes, clock state, and Net-Hunter processing activity belong to separate append-oriented journal domains.

### Retention is controlled destruction

> **Expiration eligibility is not permission to delete.**

Stronghold applies retention independently to authoritative PCAP and each journal domain. Holds override ordinary expiration. Destruction is explicit, attributable, and journaled.

> **A full disk is a failure condition, not permission to rewrite history.**

### HA claims remain separate

> **High availability of forwarding does not imply uninterrupted packet-history continuity.**

Forwarding availability, session/state continuity, capture continuity, and historical durability are separate claims. Stronghold records the actual result of each rather than treating successful failover as proof that nothing was lost.

### Stronghold controls the appliance lifecycle

> **Stronghold owns the operating-system and appliance release lifecycle.**

Stronghold FW uses Arch Linux as a platform foundation, but a supported Stronghold appliance is not a general-purpose customer-managed rolling Arch installation. Net-Hunter similarly treats FreeBSD, ZFS, jails, and Stronghold applications as appliance components with explicit qualification, compatibility, migration, and rollback boundaries.

## Common Stronghold Truth Separations

The architecture repeatedly preserves distinctions such as:

```text
route available
!=
route authorized

certificate valid
!=
peer authorized

packet not decoded
!=
packet not observed

no journal record
!=
event did not occur

timestamp precision
!=
timestamp accuracy

policy allowed
!=
forwarding succeeded

object expired
!=
destruction authorized

bytes received
!=
history durably committed

archived
!=
destroyed

new interpretation
!=
new historical observation

forwarding HA succeeded
!=
capture continuity guaranteed

virtual cluster identity
!=
physical capture identity

software installed
!=
Stronghold update validated
```

These distinctions are not wording preferences; they are architectural truth boundaries.

## Capture Invariant #1 — Wireshark-Class Interface Visibility

> **If a frame or packet is presented to a configured Stronghold physical interface and is observable through the supported NIC/driver capture path, Stronghold records it whether or not Stronghold recognizes, decodes, bridges, routes, or firewall-processes it.**

Physical-interface frame observation is a product invariant of the enforcement appliance itself.

The authoritative observation point is the configured supported physical interface. Wireshark/dumpcap on the same supported physical interface under the same conditions is the reference visibility baseline.

When presented to the interface, capture scope includes ordinary IPv4/IPv6 traffic and observable Layer-2/control-plane traffic such as:

```text
ARP
DHCP / BOOTP
DHCPv6
CDP
LLDP
STP / RSTP / MSTP
LACP
802.1X / EAPOL
OSPFv2 / OSPFv3
VRRP
IGMP
IPv6 NDP
802.1Q VLAN traffic
unknown EtherTypes
unknown IP protocols
vendor-specific frames
malformed traffic
```

Protocol recognition is not a prerequisite for capture. Stronghold must not claim visibility into frames that topology, NIC hardware, hardware filtering/offload behavior, or the driver did not present to the supported capture path.

Stronghold must remain truthful about NIC filtering, driver behavior, VLAN/checksum offloads, aggregation/coalescing, hardware timestamping, capture-ring/kernel drops, storage failures, and other conditions that can alter or prevent observation.

## System Topology

```text
                         MANAGEMENT NETWORK
                       ┌──────────┴──────────┐
                       │                     │
                       ▼                     ▼
               Stronghold FW          Stronghold Net-Hunter
                   MGMT                      MGMT

   PRODUCTION NETWORK
          │
          ▼
 ┌────────────────────────────┐
 │       STRONGHOLD FW        │
 │ Observe / Preserve         │
 │ Authorize                  │
 │ Bridge / Route / NAT       │
 │ Enforce                    │
 │ Journal                    │
 │ Arch Linux                 │
 │ OS SSD / Btrfs             │
 │ NVMe HOT / XFS             │
 │ SSD WARM / XFS             │
 └─────────────┬──────────────┘
               │ dedicated history network
               │ 10 / 25 / 40 GbE
               │ mTLS + explicit peer auth
               ▼
 ┌────────────────────────────┐
 │  STRONGHOLD NET-HUNTER     │
 │ Receive / Verify / Commit  │
 │ Preserve / Reprocess       │
 │ Correlate / Hunt / Query   │
 │ Export / Retain / Archive  │
 │ Journal                    │
 │ FreeBSD / ZFS / Jails      │
 │ NVMe / SAS SSD / SAS HBA   │
 └────────────────────────────┘
```

## Stronghold FW

### Platform direction

```text
Platform:          Arch Linux
Architecture:      x86_64 / amd64
Administration:    CLI first
NICs:              physical PCIe Ethernet with qualified Linux drivers
Capture:           continuous full-packet capture
Initial source:    AF_PACKET / TPACKET_V3
Format:            PCAPNG
OS storage:        dedicated local SSD / Btrfs direction
PCAP HOT storage:  NVMe / XFS direction
PCAP WARM storage: local SSD / XFS direction
```

### Dataplane targets

```text
1 GbE   current target
10 GbE  primary performance target
40 GbE  future hardware target only
```

Performance claims require representative validation. History-link speed does not establish firewall dataplane performance.

## FW Networking Model

Stronghold does not require one global forwarding mode.

### Layer-2 transparent bridge

Stronghold may bridge Ethernet/VLAN traffic without becoming the IP gateway. Capture and policy remain active.

### Layer-3 routed firewall

Stronghold may terminate IPv4/IPv6 networks, route between them, and apply stateful policy.

### Router on a stick

Stronghold may terminate multiple 802.1Q VLANs on one physical trunk and route/firewall between them. Traffic may enter and leave the same physical trunk under different VLAN contexts.

### Hybrid deployment

Different interfaces, VLANs, and bridge domains may use different forwarding models on the same appliance.

```text
WAN1             Layer 3 routed
LAN-TRUNK        Layer 3 / router on a stick
TRANSIT-A/B      Layer 2 transparent bridge
MGMT             management only
HISTORY          Net-Hunter history only
HA               cluster coordination only
```

## FW Network Objects

Stronghold uses explicit first-class objects rather than requiring administrators to encode network meaning in Linux interface names or raw nftables statements.

Expected object families include:

```text
physical interface
logical interface
VLAN
bridge domain
zone
host
network
address group
service
service group
FQDN
FQDN group
route
security policy
NAT policy
WAN preference policy
HA cluster
HA node
virtual forwarding identity
```

A VLAN preserves Stronghold object identity, numeric 802.1Q VLAN ID, description/context, and parent/bridge/routed relationships. A logical interface is a Layer-3 termination point. A bridge domain is an explicit Layer-2 forwarding context.

## Zones and Interface Roles

Zones represent security boundaries. Interfaces/VLANs represent network attachment. One routed logical interface belongs to one security zone; one zone may contain multiple logical interfaces/VLANs. Bridge-domain membership and zone membership are distinct.

Current interface roles:

```text
PRODUCTION
WAN
MANAGEMENT
HISTORY
HA
```

- `PRODUCTION` carries ordinary bridged/routed traffic.
- `WAN` carries production WAN traffic and may participate in health, routing preference, NAT identity, and adaptive selection.
- `MANAGEMENT` is for administration and approved management services, not production transit.
- `HISTORY` is for FW↔Net-Hunter history/control transfer, not production transit or normal interactive administration.
- `HA` is for cluster heartbeat/control and state/config synchronization, not production transit, history transfer, or normal interactive administration.

Configuration validation must reject unsafe combinations such as a production default route through HISTORY/HA, placing MANAGEMENT/HA in a production bridge domain, or using HISTORY as an alternate WAN.

## FQDN Policy Direction

Stronghold distinguishes DNS-backed FQDN association from directly observed Layer-7 hostname identity.

DNS-backed policy may maintain TTL-aware address sets, but an address associated with a name is not proof a particular connection used that hostname. Future Layer-7 observation may use facts such as TLS SNI, HTTP Host, DNS history, or other protocol-specific names and must record the source of the hostname fact.

## Policy and Authorization Model

### Default deny

Stronghold is explicit-allow/default-deny for Layer-2 forwarding, Layer-3/4 forwarding, and traffic destined to the appliance itself. DROP/REJECT does not suppress authoritative ingress capture.

### Rule evaluation

Rules use dense mutable integer positions, lowest to highest, first match wins. Inserting/moving shifts the affected range while preserving a dense list.

A **Policy ID is stable reference identity only** and never determines priority/order.

Historical decisions preserve policy ID/name, rule position at decision time, configuration generation, and action.

### Observation, required facts, and authorization-first processing

Stronghold's semantic processing model is:

```text
PACKET / FRAME ARRIVES
        │
        ├──────────────► CAPTURE / OBSERVE / PRESERVE
        │
        ▼
ESTABLISH REQUIRED POLICY FACTS
        │
        ▼
EARLY INGRESS AUTHORIZATION
        ├── explicit DENY ───────────────► DROP
        ├── no permitting match ─────────► DROP
        └── authorized to proceed
                    ↓
              BRIDGE / ROUTING
                    ↓
          WAN / NAT / STATE /
       REQUIRED DEEPER PROCESSING
                    ↓
               FINAL EGRESS
                    ↓
           JOURNAL ACTUAL RESULT
```

The existence of a route or NAT rule never grants permission.

Where implementation realities require a route/FIB lookup to establish a classification fact, that lookup may occur, but a successful lookup does not itself constitute authorization.

Layer-7 policy may use an `authorized_to_continue_processing` state without claiming final allow before the required Layer-7 fact is established.

### Explicit downstream states

Stronghold records not only what it did but also when later processing never occurred.

Example early denial:

```text
Observed:        YES
Authorization:   DENY
Routing:         NOT_PERFORMED
WAN selection:   NOT_PERFORMED
NAT:             NOT_PERFORMED
Final result:    DROP
```

Example authorization succeeded but routing failed:

```text
Observed:        YES
Authorization:   ALLOW
Routing:         FAILED
Reason:          NO_ROUTE
WAN selection:   NOT_PERFORMED
NAT:             NOT_PERFORMED
Final result:    DROP
```

## Routing Architecture

Only authorized traffic enters normal routing. Routing answers where authorized traffic should go, never whether it should be allowed.

Route selection:

```text
1. longest-prefix match
2. equal-prefix route-source preference:
       STATIC
       CONNECTED
       DEFAULT
       DYNAMIC
3. health / availability
4. metric
5. deterministic tie-break
```

A `/32` host route wins over a `/24` containing it. Route-source preference applies only after prefix specificity.

Initial direction includes connected, static, default, IPv4, and IPv6 routes. Dynamic routing remains later work. VRF is future-compatible but not an initial requirement; routing-domain boundaries should be preserved in the model.

A policy ALLOW with no route is a routing failure and final DROP, not a policy denial.

## Multi-WAN Architecture

Stronghold uses policy-defined WAN preference rather than generic load balancing.

A destination/service policy defines explicitly eligible WANs and preferred WAN. Scheduled/adaptive selection may choose only from the allowed set. A physically available WAN is never assumed eligible.

Adaptive measurements may include reachability, RTT, loss, and jitter with hysteresis/hold-down. Degradation is destination/service specific rather than automatically declaring an entire WAN bad. Existing sessions normally remain bound to their existing WAN/NAT identity.

Traffic Decision Journal history preserves preferred WAN, eligible set, selected WAN, reason, path state, configuration generation, route, and NAT decision.

## NAT Architecture

NAT is separate from security authorization and never grants permission by itself.

Initial intended capabilities include:

```text
masquerade
static SNAT
DNAT / port forwarding
static 1:1 NAT
NAT exemption / no-NAT
```

Original and translated tuples remain distinct. NAT follows selected route/WAN/state. IPv6 translation features remain deliberate future decisions.

## Configuration Management

Stronghold owns appliance configuration through candidate configuration, validation, diff, commit, and monotonic configuration generations.

```text
RUNNING
  ↓ copy/edit
CANDIDATE
  ↓ validate + diff
COMMIT
  ↓
NEW GENERATION
```

Validation includes whole-system semantics: addressing, VLAN relationships, object references, policy/NAT/WAN relationships, MANAGEMENT/HISTORY/HA role restrictions, routing consistency, rule positions, and other hard architecture boundaries.

Every successful commit creates a new monotonically advancing generation. Rollback restores older content by creating a new generation; history never moves backward.

Risky remote changes support commit-confirmed protection. Successful committed generations are intended to be preserved in the isolated Net-Hunter FW Configuration Backup Jail.

Configuration generation, configuration schema version, software release version, journal schema versions, and HA protocol/state format are distinct identities. They must not be collapsed into one version number merely for convenience.

## Stronghold FW High Availability Architecture

### Initial HA model

Stronghold anticipates optional **active/standby** FW clustering. Active/active forwarding is not the initial HA model because it materially increases session ownership, NAT, asymmetric routing, duplicate observation, journal ordering, and capture-authority complexity.

```text
                   ┌────────────────┐
                   │ Stronghold FW-A│
                   │    ACTIVE      │
                   └───────┬────────┘
                           │
                    HA CONTROL/STATE
                           │
                   ┌───────┴────────┐
                   │ Stronghold FW-B│
                   │    STANDBY     │
                   └────────────────┘
```

### Node identity and cluster identity

Each node retains its own:

```text
Stronghold Appliance ID
hostname / management identity
physical NIC and MAC identity
history-link identity
HA-link identity
node certificate identity
local storage
clock state
source journals
capture provenance
```

The pair has a stable **Stronghold Cluster ID**.

Cluster-owned **network forwarding identity** is separate from node identity. Routed gateway addresses, static WAN addresses, and virtual MACs where required belong to the cluster and are claimed only by the ACTIVE node.

> **Virtual network identity is forwarding identity, not capture identity.**

Historical packet provenance always names the physical Stronghold appliance/interface that actually observed the traffic.

### Routed/VLAN HA

For routed interfaces and router-on-a-stick VLANs, clients use cluster-owned virtual gateway identities. Example:

```text
VLAN 10 USERS   → cluster gateway 10.10.10.1
VLAN 20 SERVERS → cluster gateway 10.20.20.1
VLAN 30 VOICE   → cluster gateway 10.30.30.1
```

The active node owns/responds for those virtual identities. On failover, the new active node assumes them and should announce/re-establish L2 neighbor ownership using the applicable IPv4/IPv6 mechanisms.

Static/transferable WAN identity follows the same cluster-ownership model, which is important for NAT/session continuity.

Dynamic WAN identity such as DHCP/PPPoE requires a later protocol-specific HA contract because lease/session/MAC binding behavior can differ by provider.

### Transparent Layer-2 HA

Transparent bridge deployments have no required virtual gateway IP. Both nodes may be physically connected to the outside and inside switching domains, but only one bridge path may forward.

```text
FW-A bridge: ACTIVE/FORWARDING
FW-B bridge: STANDBY/BLOCKED
```

Failover transfers explicit bridge ownership. Stronghold must prevent simultaneous forwarding paths that would create an L2 loop.

### Cluster versus node configuration

Configuration is split conceptually into:

```text
CLUSTER CONFIGURATION
    VLANs/zones/bridge domains
    security policy
    NAT
    routes/WAN preference
    virtual forwarding identities
    retention/cluster settings

NODE-LOCAL CONFIGURATION
    Appliance ID
    management address
    history address
    HA address/link
    physical NIC/PCI mappings
    local storage mapping
    node certificate identity
```

Effective node configuration is cluster configuration plus node-local configuration.

### Cluster commit synchronization

Normal cluster commit should conceptually:

```text
candidate
  ↓
validate cluster semantics
  ↓
validate FW-A hardware/node context
  ↓
validate FW-B hardware/node context
  ↓
stage generation on both
  ↓
coordinated activation
  ↓
generation ACTIVE
```

A standby with stale configuration is a degraded failover target.

When a peer is unavailable, normal commit may be blocked or an explicitly privileged **degraded commit** may be permitted according to the later HA configuration contract. Any degraded commit must expose and journal the resulting configuration mismatch rather than pretending the pair is synchronized.

Configuration synchronization state is an explicit input to failover eligibility. Exact behavior for automatic promotion under stale configuration remains to be frozen; the architecture leaves room for `STRICT` versus availability-oriented behavior without silently ignoring mismatch.

Failover itself is a runtime state transition and does **not** create a new configuration generation when the same synchronized generation remains active.

### Session/state synchronization

The active node synchronizes operational state needed for stateful failover, including where applicable:

```text
stateful firewall sessions
NAT mappings
session timers
selected WAN
route/session binding
virtual identity/runtime state
future Layer-7 state where practical and justified
```

State synchronization uses an advancing sequence/health context so Stronghold can distinguish current state from lagged or unknown state.

Expected health concepts include:

```text
IN_SYNC
MINOR_LAG
DEGRADED
OUT_OF_SYNC
UNKNOWN
```

Stronghold may preserve synchronized sessions when the external network identity and protocol permit it. It must not promise universal seamless session survival across every failover.

### HA control versus bulk synchronization

Heartbeat/control and bulk session/state synchronization are logically separate traffic classes even when they initially share one physical HA interface.

HA heartbeat/control is lightweight and high priority. Bulk state replication may throttle or lag and must not starve heartbeat/control. Capture remains the highest live-data priority; HA state synchronization can fall behind under extreme load, but control traffic must have enough reserved capacity to avoid false failure decisions caused by state-sync congestion.

### Heartbeat content and ordering

A heartbeat should eventually carry enough cluster state to establish facts such as:

```text
Cluster ID
Node Appliance ID
HA role
heartbeat sequence
configuration generation
state-sync sequence
system health
production-interface readiness
WAN readiness summary
clock state
software / HA protocol compatibility state
```

Exact interval/timeout values are an implementation/hardware-validation contract and are not frozen here.

### Loss of HA communication

> **Loss of peer communication is not, by itself, proof that the peer is dead.**

One lost heartbeat path or failed HA cable does not automatically authorize the standby to become ACTIVE. The cluster enters a degraded peer-control state and uses the future fencing/election contract to decide whether promotion is safe.

A secondary peer-observation path, such as MANAGEMENT, may help distinguish a failed HA cable from a failed peer, but loss of management reachability also does not independently prove death.

### Promotion gate

Conceptually, standby promotion evaluates:

```text
local node healthy?
required network paths sufficiently ready?
peer unavailable according to HA decision logic?
fencing/election ownership safe?
configuration state acceptable?
software/protocol compatibility acceptable?
state-sync condition known?
        ↓
PROMOTE
```

No single heartbeat timeout bypasses those conditions.

### Split-brain prevention and fencing

> **A node may claim ACTIVE cluster identity only after satisfying the cluster fencing/election contract.**

Stronghold must not allow two nodes to casually claim the same virtual IP/MAC/bridge identity simultaneously. When ownership cannot be established safely, preserving single-owner safety takes precedence over aggressive failover.

Exact fencing/election technology remains unfrozen. Potential future mechanisms may include peer election, multiple network observations, power/switch fencing, or an optional independent witness.

**Net-Hunter is not a required HA witness or quorum dependency.** Hunter failure must remain independent of FW forwarding availability.

### Node states and planned operation

The architecture anticipates explicit node states such as:

```text
ACTIVE
STANDBY
PROMOTING
DEMOTING
MAINTENANCE
VALIDATING
DEGRADED
FAILED
```

Controlled manual failover should validate standby health, configuration synchronization, state synchronization, software/HA compatibility, and required interfaces before transition. `MAINTENANCE` makes a node intentionally ineligible for automatic promotion while administrators perform updates or hardware work.

Planned/manual transitions and failure-driven transitions remain distinguishable in journal history.

### Capture continuity and local history

HA synchronization is **not** a mandatory real-time duplicate PCAP stream.

The ACTIVE FW follows the normal capture path:

```text
qualified physical interface
  ↓
local durable capture
  ↓
verified Net-Hunter handoff
```

The standby synchronizes operational state, not every captured packet.

Each node records only traffic actually presented to that node's qualified capture path. If FW-A fails with untransferred history on local storage, that history remains FW-A history. On recovery, closed or recoverable segments are reconciled and sent to Hunter where possible.

An open segment interrupted by crash/power loss is recovered/finalized or marked unrecoverable according to the future segment-recovery contract. Stronghold does not silently discard it simply because it was open.

Successful forwarding failover does not imply zero packet loss or zero capture gap. Any observed gap remains explicit.

### HA journaling

HA runtime state belongs primarily in the System/Health Journal; human-triggered HA changes also belong in the Administrative Journal.

Events should distinguish, where applicable:

```text
HA_PEER_DISCOVERED
HA_PEER_LOST
HA_LINK_DEGRADED
HA_STATE_SYNC_LAG
HA_STATE_OUT_OF_SYNC
HA_CONFIG_OUT_OF_SYNC
HA_VERSION_INCOMPATIBLE
HA_PROMOTION_STARTED
HA_PROMOTION_BLOCKED
HA_FENCING_FAILED
HA_SPLIT_BRAIN_DETECTED
HA_ROLE_CHANGED
VIRTUAL_IDENTITY_ASSUMED
HA_FAILOVER_COMPLETE
HA_MANUAL_FAILOVER
HA_MAINTENANCE_ENTERED
```

Journal facts preserve Cluster ID, physical node Appliance ID, role transition, configuration generation, sync state/sequence, software/protocol compatibility, reason, and result where applicable.

## Stronghold-Controlled Appliance Lifecycle and Updates

### Stronghold FW appliance ownership

Stronghold FW is built on Arch Linux but is operated as a Stronghold appliance.

Customers should receive Stronghold-qualified releases rather than unrestricted general Arch updates. The appliance release contract should control/qualify the versions and compatibility of components such as:

```text
Stronghold software
Linux kernel
NIC drivers
managed firmware where applicable
nftables / netfilter
AF_PACKET / AF_XDP and capture stack
system libraries required by Stronghold
configuration schema
journal/schema versions
HA protocol version
HA state-sync serialization format
```

Routine administrator use of unrestricted `pacman -Syu` is outside the supported appliance lifecycle.

### Release identity and verification

A Stronghold release must have a stable release identity and integrity/authenticity information. Update material must be cryptographically verified before activation.

Stronghold should support:

```text
controlled online retrieval
signed offline update bundles
```

so restricted/isolated deployments do not require public-Internet connectivity to remain maintainable.

Exact signing hierarchy, release metadata, and key-rotation/recovery mechanics remain to be frozen.

### Update preflight

Before activation, Stronghold should establish the update's actual prerequisites and risks, including as applicable:

```text
current/target Stronghold version
release signature/integrity state
configuration generation/schema
journal/state schema compatibility
HA protocol/state-format compatibility
peer HA health and synchronization
local OS/filesystem health
capture storage health
available update/rollback space
physical interface/NIC mapping
```

An HA node already in a failed or dangerously degraded state is not treated as a routine safe rolling-update starting point.

### Controlled lifecycle

The intended lifecycle is:

```text
SIGNED / VERIFIED RELEASE
        ↓
PREFLIGHT
        ↓
COMPATIBILITY CHECK
        ↓
CHECKPOINT / SNAPSHOT WHERE APPLICABLE
        ↓
INSTALL
        ↓
REBOOT IF REQUIRED
        ↓
STRONGHOLD VALIDATION
        ↓
READY / STANDBY_READY / ACTIVE_READY
```

A successful package install or successful operating-system boot is not enough to claim update success.

### Capture/enforcement regression qualification

Updates that can materially affect packet visibility or enforcement require appropriate regression qualification, especially changes involving:

```text
Linux kernel
NIC drivers
NIC firmware
nftables
netfilter
AF_PACKET
AF_XDP
hardware/software offloads
capture/storage path
```

The update must not be called validated merely because services started. Stronghold must re-establish the relevant capture, interface, firewall, routing, journal, storage, and HA readiness facts for the appliance role.

### Configuration/schema migration

Configuration migration is a controlled transition, not an in-place opaque rewrite.

```text
source config + schema
        ↓
migration candidate
        ↓
validate target schema
        ↓
activate target state
```

The source configuration/schema must remain recoverable until the new state is successfully validated according to the rollback contract.

Software version, configuration schema, journal/index schema, and HA state format remain separate compatibility dimensions.

### Rollback

Btrfs snapshots/checkpoints may assist FW rollback, but a filesystem snapshot alone is not a complete rollback contract.

Rollback must account for:

```text
boot/kernel state
Stronghold software version
configuration schema
journal/runtime state formats
HA protocol/state compatibility
managed firmware implications where applicable
```

An older binary must not simply be started against newer incompatible state and be called a successful rollback.

### HA rolling update

The preferred active/standby update sequence is:

```text
FW-A ACTIVE
FW-B STANDBY
      ↓
upgrade FW-B
      ↓
validate FW-B
      ↓
FW-B STANDBY_READY
      ↓
controlled failover
      ↓
FW-B ACTIVE
FW-A STANDBY
      ↓
validate production state
      ↓
upgrade FW-A
      ↓
validate cluster health
```

Mixed-version operation is temporary and permitted only inside an explicitly supported upgrade compatibility window.

During mixed-version operation, peers must negotiate/use only compatible HA protocol/state representations. Newer state must not be silently partially parsed or discarded by the older peer.

> **A failed standby update stops the rolling process. Stronghold must not automatically risk the remaining healthy active node after the peer update fails.**

A validation window between first-node promotion and second-node upgrade may be operator-controlled so the old node remains a viable failback target until the new release has demonstrated acceptable production behavior.

### Standalone FW update

A standalone FW update that requires reboot/service interruption creates a real forwarding/capture outage.

Before planned interruption, Stronghold should finalize active capture segments where possible, complete required durability/journal transitions, and record the planned maintenance/update state.

Stronghold must report the actual outage rather than describe standalone maintenance as hitless or highly available.

### Update journaling

Update transitions belong in Administrative and System/Health Journals as applicable, including facts such as:

```text
UPDATE_STARTED
UPDATE_BUNDLE_VERIFIED
UPDATE_PREFLIGHT_FAILED
UPDATE_INSTALL_COMPLETE
UPDATE_REBOOT_REQUIRED
UPDATE_BOOTED
UPDATE_VALIDATION_STARTED
UPDATE_VALIDATION_FAILED
UPDATE_VALIDATION_SUCCEEDED
UPDATE_ROLLBACK_STARTED
UPDATE_ROLLBACK_SUCCEEDED / FAILED
NODE_STANDBY_READY
```

Records preserve source/target release identities, configuration/schema state, administrator/process authority, signature/integrity result, reason, and outcome where applicable.

## Administrative Identity, Authentication, and Authorization

Stronghold separates authentication, authorization, and journaled administrative history.

Remote authentication direction:

```text
Active Directory via LDAPS only
RADIUS
TACACS+
```

Plain LDAP is not supported for AD authentication and there is no LDAPS→LDAP downgrade. LDAPS server certificate validation is required. Multiple DC/LDAPS endpoints may be configured with explicit health/failover behavior.

Protected local identity remains for installation, physical-console recovery, and break-glass use. External authentication failure must not stop the dataplane.

Authorization is role-based. Important permissions remain separable, including candidate edit, validate, commit, rollback, network/security/system administration, history hunt/view/export, journal audit, retention/hold/destruction authority, HA administration, and appliance update administration.

Net-Hunter access does not imply FW administration. Normal remote administration enters through MANAGEMENT; HISTORY and HA are not normal interactive administration paths.

## Appliance Identity, PKI, and FW↔Hunter Trust

Each appliance has a stable Stronghold Appliance ID independent of hostname, IP address, management address, or current certificate.

Stronghold favors a Stronghold-specific appliance trust hierarchy. History transport uses mTLS with no plaintext fallback.

A valid certificate establishes cryptographic identity but does not authorize a FW to use every Hunter. Peer authorization is explicit and revocable. Certificate rotation preserves Appliance ID. Expiry/revocation/peer rejection blocks history transfer and creates backlog/degraded state rather than stopping the dataplane.

Management/UI certificates and history-transfer certificates are separate purposes. TLS does not replace segment identity, hash, independent Hunter verification, durable commit, or acknowledgement.

HA peers likewise require authenticated explicit cluster membership; a system merely connected to the HA network cannot assert cluster role or inject trusted state.

## Time, Clock Authority, and Timestamp Truthfulness

Stronghold stores authoritative wall-clock time in UTC. Timezone is presentation only.

Critical journal ordering is independent of wall-clock correctness through advancing sequence/order and monotonic time where appropriate. A backward wall-clock adjustment cannot reverse journal causality.

Initial synchronization supports multiple configured NTP sources. NTS, PTP, and hardware timestamping remain future capabilities where justified.

Clock states include concepts such as:

```text
SYNCHRONIZED
HOLDOVER
UNSYNCHRONIZED
CLOCK_FAULT
```

Clock corrections/source transitions are Time/Clock Journal events.

> **Timestamp precision must never be presented as timestamp accuracy.**

Net-Hunter preserves source FW observation time separately from Hunter receipt/verification/commit/processing time.

## Packet Authority and Journal Architecture

### Packet authority

Raw PCAPNG is authoritative for what network traffic Stronghold observed. Segment identity, capture provenance, packet/byte/drop accounting, and integrity metadata establish packet-history lineage. Missing decoded data never proves the packet did not exist.

### Derived interpretation

Net-Hunter may produce derived information such as:

```text
flow/session reconstruction
protocol decoding
DNS / hostname correlation
Layer-7 interpretation
asset relationships
behavioral interpretation
detection results
threat-intelligence enrichment
future analytical results
```

Derived data remains traceable to the authoritative PCAP/journal/configuration source. Reprocessing produces new or superseding derived interpretation without modifying the source observation.

The UI may present the newest valid derived interpretation by default while preserving lineage and the fact that earlier processing may have been partial or different.

### Separate journal domains

Stronghold defines separate append-oriented domains:

```text
Administrative Journal
System / Health Journal
Traffic Decision Journal
Trust / Identity Journal
Time / Clock Journal
Hunter Processing Journal
```

Committed entries are not silently modified. Corrections, reprocessing, recovery, retries, retention/destruction, update/rollback results, or later success create new entries.

Traffic Decision Journal preserves authorization, policy, route, WAN, NAT, bridge/routing disposition, final disposition, reason, and configuration generation. `NOT_PERFORMED` is explicit where later pipeline work did not occur.

System/Health includes interface/NIC, capture drops, storage pressure, Hunter availability, route/WAN state, service/jail/ZFS state, HA state, and update validation state.

Trust/Identity includes appliance enrollment, FW↔Hunter authorization, certificate lifecycle, peer rejection/revocation, authentication trust failures, and role/trust-map changes.

Time/Clock includes synchronization/source/offset/holdover/fault transitions.

Hunter Processing appends receipt, verification, commit, indexing, correlation/reprocessing, export, retention/archive, and processing failure/retry history without replacing originating FW history.

Critical journal storage should support cryptographic append/tamper verification. Exact schemas, UUID, hash/signature, and finalization contracts remain to be frozen. PCAP integrity remains segment-oriented, not per-packet chained.

## Retention, Holds, Archive, and Controlled Destruction

Authoritative PCAP and each journal domain have independent retention policy. Values/classes are deployment policy, not hard-coded architecture defaults.

Retention classes may attach to explicit scope such as appliance, site, VLAN, zone, or other approved administrative context. Destructive retention must not depend on speculative parser-derived classification without an explicit later contract.

Conceptual lifecycle:

```text
CREATE
  ↓
ACTIVE
  ↓
RETAINED
  ↓
ELIGIBLE_FOR_EXPIRATION
  ↓
AUTHORIZED_DESTRUCTION
  ↓
DESTROYED
  ↓
DESTRUCTION JOURNALED
```

Legal, investigative, and administrative holds override ordinary expiration. Creating/changing/releasing a hold is an Administrative Journal event.

Hold scope should favor reproducible facts such as appliance, time range, VLAN, zone, IP/host, MAC, capture segment, or explicit journal range rather than unstable parser-dependent semantics where destructive behavior could change when interpretation improves.

Manual destruction is distinct from normal retention-driven destruction and requires explicit privileged authority and reason. Destruction never erases the historical fact that the object existed; surviving journal history preserves applicable identity, source, time range, integrity identity/hash where appropriate, policy/reason, authority, time/order, and result.

Archive is a lifecycle transition, not destruction, and preserves identity/integrity/lineage/retrieval state.

## Local FW Storage Tiers and Pressure

```text
RAM        buffer only
  ↓
NVMe/XFS   HOT authoritative PCAP ingest
  ↓
SSD/XFS    WARM / outage-backlog
  ↓
Net-Hunter dedicated history network
```

Long-term HDD/RAID history belongs to Net-Hunter. PCAP storage is separated from the OS filesystem.

FW pressure states include:

```text
NORMAL
HIGH
URGENT
CRITICAL
```

Pressure may throttle/pause secondary work, accelerate safe transfer/tiering, and advance only already-authorized expiration. It does not silently create destructive authority.

Acknowledged history that Hunter independently verified and durably committed is safer to expire locally than unacknowledged history. By default, unacknowledged authoritative history is not automatically deleted merely to hide storage pressure.

If storage exhaustion prevents durable capture, Stronghold reports a real history gap while forwarding/enforcement may continue if healthy.

## Net-Hunter Outage Behavior

Hunter unavailability does not determine whether FW capture, bridge/routing, NAT, or enforcement continues. FW accumulates finalized local backlog while capacity permits and exposes pending volume/oldest pending/HOT/WARM state. Ultimate exhaustion follows truthful gap behavior rather than silent discard.

Planned Hunter maintenance/update is distinguished from unexpected failure, but both pause verified handoff when ingest is unavailable and cause FW backlog to grow locally.

## Capture Segment Lifecycle

Intended local lifecycle remains:

```text
CAPTURING
    ↓
CLOSED
    ↓
SYNCED
    ↓
HASHED
    ↓
CATALOGED
    ↓
VERIFIED
    ↓
TRANSFER_ELIGIBLE
```

Only finalized closed segments transfer to Hunter. A source segment is not removed merely because a network copy completed.

## Dedicated Stronghold History Network and Handoff

History transfer uses a dedicated 10/25/40 GbE path as deployed. This link is not production transit or normal administration.

Verified handoff:

```text
FW finalizes segment/source journal batch
       ↓
FW integrity metadata
       ↓
mTLS transfer to authorized Hunter ingest
       ↓
Hunter complete receive/finalize
       ↓
independent verification
       ↓
durable commit
       ↓
Hunter Processing Journal state
       ↓
verified ACK
       ↓
FW acknowledgement state
```

Hunter must never ACK unverified/uncommitted history, including under storage pressure.

## Stronghold Net-Hunter

### Platform direction

```text
Platform:       FreeBSD
Storage:        ZFS
Isolation:      jails
Memory:         large ECC RAM capacity
Fast storage:   NVMe
Warm storage:   SAS SSD as required
Bulk history:   HBA-attached SAS storage under ZFS
```

The host owns hardware, HBA/raw disk visibility, ZFS, host networking/PF, jail lifecycle, FreeBSD updates, and hardware/storage health.

### Net-Hunter jails

1. **PCAP Data Ingest Jail** — authenticated/authorized FW receive, segment/source-journal verification, commit, and ACK. No external user hunt access.
2. **Record Processing Jail** — reads authoritative PCAP/source journals, builds/enriches searchable records/indexes, and reprocesses history. It does not rewrite authoritative source history.
3. **External User Interface Jail** — hunt/query/search/timeline/correlation/view/export/report/status. Authoritative PCAP, source journals, processed history, indexes, and config history are read-only from the UI.
4. **FW Configuration Backup Jail** — isolated versioned firewall config history accessible only to authorized FW appliances and local Net-Hunter administration.

Writable UI state is non-authoritative workspace such as session/query state, derived exports/reports/notes, and download staging.

### Net-Hunter storage and pressure

```text
NVMe FAST
    ingest/index/active records+journals/query/recent history

SAS SSD WARM
    recent/frequently accessed history / processing as appropriate

HBA → SAS ZFS HISTORY
    authoritative long-term PCAP
    journal/history retention
```

ZFS protects storage but does not replace Stronghold object IDs, hashes, journal advancement, or transfer lineage. RAIDZ2 is a current bulk-history candidate; exact topology remains to be sized/validated. Hardware RAID must not hide bulk-history disks from ZFS.

Hunter should expose utilization, ingest/expiration rates, net growth, backlog, and projected capacity where estimable. Under pressure, preserve ingest/verification/commit first and reduce nonessential processing. If Hunter cannot durably commit, the ACK path fails and FW retains backlog.

### Net-Hunter lifecycle and updates

Net-Hunter update control is layered:

```text
FreeBSD host
      ↓
Stronghold host-management layer
      ↓
jails
      ↓
applications within jails
      ↓
processed-record / index schemas
      ↓
External UI
```

Host updates and application-jail updates are related but not automatically one inseparable operation. Application jails should be updatable/recoverable without granting them host/ZFS authority.

Where practical, jail/application replacement may use a prepare/validate/switch/retire pattern so a failed new application instance does not require rewriting authoritative source history.

Derived records/indexes are rebuildable from authoritative PCAP and source journals. Failed index/schema migration must not mutate authoritative packet/source-journal history merely to make a new release work.

Planned Hunter maintenance may make ingest/query temporarily unavailable; FW capture continues and local backlog grows according to the existing outage contract.

### ZFS compatibility boundary

A FreeBSD/ZFS software update does **not** automatically authorize enabling new irreversible pool features.

```text
FreeBSD / ZFS software updated
!=
permission to upgrade pool feature set
```

ZFS pool-format/feature upgrades are separate explicit compatibility operations because they may prevent rollback to an older FreeBSD/ZFS stack. Stronghold must not silently enable irreversible pool features merely because the newer software supports them.

## Resource Priority

On Stronghold FW:

```text
1. packet acquisition
2. active PCAP writes
3. segment finalization / minimum integrity
4. essential observation/decision/journal state
5. HA heartbeat/control reserved lightweight capacity
6. local tier movement/backlog
7. HA bulk state synchronization
8. transfer to Net-Hunter
9. compression where approved
10. deeper analytics outside live path
```

The exact scheduler/CPU/queue implementation remains future work. The intent is that capture remains first, heartbeat/control cannot be starved by bulk HA sync, and secondary tasks throttle rather than hiding loss.

## Engineering Completeness Test

A Stronghold feature should be judged against this operational question:

> **When this fails at 2:00 AM, will Stronghold tell the operator exactly what it observed, what it decided, why it decided it, what it actually did, and what it could not establish?**

For a feature to be complete, important failure paths should preserve enough authoritative packet/journal/configuration/state information to answer, where applicable:

```text
what was observed?
what was known?
what was unknown?
what was decided?
why?
what was actually performed?
what was NOT_PERFORMED?
what failed?
what history remains?
what was lost or could not be established?
```

## Architectural Summary

Stronghold FW owns the present:

```text
observe → preserve → establish required facts → authorize → bridge/route/NAT → enforce → journal → hand off verified history
```

Stronghold Net-Hunter owns the past:

```text
receive → verify → durably commit → preserve → journal → process/reprocess → correlate → hunt → query → export/retain/archive
```

Stronghold preserves distinct authority for:

```text
authoritative PCAP packet history
authoritative operational / decision journals
derived interpretation
physical node capture provenance
cluster forwarding identity
source FW journals
Hunter processing journals
runtime policy/routing/NAT/WAN/HA decisions
configuration generations and schemas
software / update lifecycle state
retention/hold/destruction state
user-generated notes/views/exports
```

Later implementation phase sequencing remains intentionally unfrozen while the complete product architecture is still being defined.
