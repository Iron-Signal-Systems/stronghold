# Stronghold Go Engineering Standard

## Purpose

This file defines the Go implementation standard for Stronghold.

Stronghold Go code follows the same Iron Signal Systems engineering direction used by FI and Guidon: explicit, focused, reviewable code with strong native-platform behavior and minimal unnecessary abstraction.

The Go module is expected to live under:

```text
/go
```

Current approved Go baseline:

```text
Go 1.27.0
```

## Repository Layout

Prefer focused packages under `go/internal/` representing real responsibilities.

Expected directions include:

```text
internal/capture/
internal/pcapng/
internal/catalog/
internal/storage/
internal/tiering/
internal/compression/
internal/offload/
internal/network/
```

Do not create catch-all packages such as `util`, `common`, `helpers`, `framework`, `platform`, or `misc` merely to hold unrelated functionality.

Do not create packages for future systems before the current roadmap phase requires them.

## Source Licensing

Every Go source file must contain the Stronghold source-review header:

```go
// Copyright (c) 2026 John Joseph Wood. All rights reserved.
// Use of this source code is governed by the Stronghold
// Source Review License, Version 1.0, found in the repository root LICENSE file.
```

Scripts, Makefiles, and other applicable source artifacts should carry the equivalent appropriate notice.

## General Coding Style

Prefer obvious code over clever code.

Use `switch` for discrete alternatives, classifications, and state handling.

Use `if` for guards, ranges, boolean conditions, and compound predicates.

Keep control flow shallow where practical.

Prefer early validation and explicit return paths.

Avoid unnecessary indirection and unnecessary interfaces.

Introduce an interface only when a real implementation boundary, interchangeable implementation, or test boundary justifies one.

Do not introduce generic factories, registries, dependency-injection frameworks, reflection-driven behavior, generalized callback systems, or abstract plugin mechanisms unless the actual current requirement needs them.

## Function and Type Organization

Prefer a consistent source-file order such as:

```text
constants
sentinel errors
types
exported functions
internal functions
```

Within each logical section, keep names in predictable alphabetical order where doing so does not break necessary semantic grouping.

Keep tightly coupled lifecycle functions together when separating them solely for alphabetization would make the code harder to follow.

Export only what another package actually needs.

Do not create oversized files containing unrelated responsibilities merely to reduce file count, and do not fragment simple packages into excessive one-function files without a real responsibility boundary.

## Dependencies

Prefer the Go standard library.

For Linux-native interfaces, prefer `golang.org/x/sys/unix` when it exposes the required API correctly.

Add third-party dependencies only when they provide a concrete capability that would otherwise require significant custom code or security-sensitive reimplementation.

Do not add frameworks for convenience.

Every new dependency should be reviewable for:

```text
purpose
maintenance status
license
security impact
transitive dependency cost
supported Linux behavior
```

## CGO

Prefer pure Go and Linux-native syscall/API bindings.

Do not introduce CGO merely because a C library is convenient.

CGO may be used only when a required capability cannot reasonably be implemented through Go/native Linux interfaces and the dependency is justified.

Any CGO introduction requires explicit review of build portability, memory-safety boundaries, deployment dependencies, performance impact, and security impact.

## Linux Platform Targeting

Stronghold initially targets:

```text
OS:   Linux / Arch Linux
ARCH: amd64 / x86_64
```

Design Linux-owned code for Linux. Do not weaken or abstract away useful Linux behavior merely so the same implementation can compile on Windows or FreeBSD.

Compile-time portability is not more important than truthful behavior.

Use Linux-native facilities where appropriate, including packet sockets, memory mapping, netlink, filesystem durability primitives, cgroups/systemd resource controls, and later nftables integration.

Do not replace an appropriate native or structured Linux interface with parsing shell command output merely for convenience.

External commands are acceptable only when the command itself is the appropriate supported interface or no suitable programmatic interface exists; such use must be intentional, bounded, and documented.

## Native Boundary Safety

Native code must validate operating-system output before turning it into Stronghold facts.

Validate, as applicable:

```text
returned lengths
buffer offsets
structure versions
structure sizes
integer ranges
file descriptors
native result codes
termination conditions
alignment
packet lengths
capture metadata
```

Do not trust a mapped ring or native buffer merely because a system call returned success.

Keep `unsafe` usage narrow and isolated to the native boundary where practical.

## Capture Engineering

The initial preferred capture source is AF_PACKET with TPACKET_V3.

Capture workers must expose truthful packet and drop accounting.

Do not apply BPF filters to the authoritative configured full-capture stream merely to satisfy downstream offload or retention selection.

Prefer one simple measurable capture path first. Add fanout, queue-specific scaling, CPU affinity, AF_XDP, or other optimizations only when profiling demonstrates a need or a later approved requirement demands them.

Capture code must not block on compression, remote offload, deep indexing, or analytics.

## Error and State Handling

Errors should identify what failed.

Wrap errors when operation context materially improves diagnosis.

Do not swallow capture loss, durability failures, integrity failures, storage failures, or cleanup failures that affect state.

Do not reduce a multi-stage operation to one generic error when Stronghold knows which stage actually occurred.

Model meaningful state explicitly. Do not use a single `Success bool` when the system knows more.

A zero value must not accidentally mean a successful capture, completed durability boundary, successful verification, or successful offload.

## Filesystem and Durability Engineering

`write()` success is not durability.

Durability-sensitive code must clearly distinguish stages such as:

```text
bytes written
file synchronized
file closed
directory synchronized
segment finalized
hash calculated
catalog recorded
destination copied
destination verified
source removed
```

Do not silently continue after `fsync`/equivalent failure, storage full, I/O error, or verification conflict.

Never delete the source capture during tier migration until the destination has crossed the required durability and verification boundary.

## Resource Priority

Stronghold's resource-priority order is:

```text
1. packet receive
2. active RAM-to-hot-tier PCAP writes
3. segment finalization / minimum integrity metadata
4. essential catalog state
5. tier migration
6. compression
7. remote offload
8. deep indexing / analytics
```

Compression, tiering, offload, and analysis workers must be designed so they can throttle or pause without blocking the capture path.

Any observed packet loss or sustained capture-ring pressure must be capable of triggering suspension/throttling of nonessential background work.

## Tests

Tests should live with the package.

Test successful behavior and failure/refusal behavior.

Capture-, integrity-, and durability-sensitive work requires explicit failure tests, including as applicable:

```text
malformed ring metadata
short packet data
packet-drop reporting
short write
storage full
fsync failure
hash mismatch
interrupted segment finalization
compression failure
destination copy failure
destination verification failure
offload unavailable
restart with pending migration
```

Tests should establish what Stronghold claims, not merely whether a function returned nil.

## Required Validation

Before a Go change is considered ready for review, run as applicable:

```text
gofmt
go test ./...
go vet ./...
```

Stronghold's Linux-native capture and durability behavior must be runtime-tested on Linux. Cross-compilation alone does not prove packet-ring behavior, packet-loss accounting, filesystem durability, cgroup priority behavior, storage-pressure behavior, or power-loss recovery.

If a required runtime check could not be executed, state that explicitly.

## Change Review

Before finishing a Go implementation slice, verify:

- code follows the current Stronghold roadmap and architecture;
- capture priority has not been weakened;
- packet-loss and degraded states remain truthful;
- Linux-native behavior is appropriate;
- no future-phase abstraction was introduced unnecessarily;
- durability and migration behavior are explicit;
- failure behavior is tested;
- formatting/tests/vetting were performed where possible; and
- intended Linux amd64 builds remain intact.
