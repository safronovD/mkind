<!--
  Sync Impact Report
  ==================
  Version change: (none) -> 1.0.0 (initial adoption)

  Principles established:
    I.   KinD Integration Strategy (DEFERRED - TODO)
    II.  Simplicity & YAGNI
    III. Test-First Development
    IV.  Kubernetes API Conventions
    V.   Hermetic & Graceful Degradation

  Added sections:
    - Core Principles (5 principles, 1 deferred)
    - Technical Constraints
    - Development Workflow
    - Governance

  Removed sections: (none - initial version)

  Templates requiring updates:
    - .specify/templates/plan-template.md ✅ aligned
    - .specify/templates/spec-template.md ✅ aligned
    - .specify/templates/tasks-template.md ✅ aligned
    - .specify/templates/checklist-template.md ✅ aligned
    - .specify/templates/commands/ (no files found, N/A)

  Follow-up TODOs:
    - TODO(KIND_INTEGRATION_STRATEGY): Decide between upstream
      dependency, fork, or go-lib approach after initial prototyping.
-->

# mkind Constitution

## Core Principles

### I. KinD Integration Strategy

TODO(KIND_INTEGRATION_STRATEGY): The relationship between mkind and
upstream KinD is under evaluation. Three options are being considered:

1. **Upstream dependency** - Use KinD as-is via git submodule or Go
   module import. Do not modify KinD internals.
2. **Fork** - Maintain a fork of KinD with multi-host extensions.
3. **Go library** - Import and extend KinD's Go packages as library
   dependencies.

This principle will be finalized after initial prototyping clarifies
which integration model best supports multi-host cluster creation,
node image reuse, and kubeadm bootstrap coordination.

**Interim constraint**: Until this principle is ratified, all work
MUST be structured so that switching between integration models
remains feasible. Avoid deep coupling to any single approach.

### II. Simplicity & YAGNI

Every design decision MUST start with the simplest viable solution.
Complexity MUST be justified against a concrete, immediate need -
not a hypothetical future requirement.

- The MVP underlay MUST use Docker bridge networks with per-host
  unique subnets and static IP routes. No tunnels, no VXLAN, no
  WireGuard, no overlay networks until the bridge approach is proven
  insufficient for a documented use case.
- kindnetd handles pod-to-pod routing, CNI configuration, and IP
  masquerading without modification. mkind solves exactly one layer:
  making kind-node container IPs routable across hosts.
- New abstractions, configuration options, or underlay modes MUST
  NOT be introduced unless the existing mechanism fails to meet a
  stated requirement.

**Rationale**: The upstream KinD project succeeds because it is
minimal. mkind inherits this philosophy. The core networking insight
(kindnetd works if the underlay is flat) eliminates the need for
complex CNI replacement in the MVP.

### III. Test-First Development

Test-Driven Development is mandatory for all feature work.

- Tests MUST be written before implementation code.
- Tests MUST fail (red) before the corresponding implementation is
  written, then pass (green), then be refactored.
- Integration tests are required for: cross-host node connectivity,
  kubeadm join operations, cluster lifecycle (create/delete), and
  static route propagation.
- Unit tests are required for: subnet allocation, cluster config
  parsing and validation, node placement logic.
- End-to-end tests MUST verify that a multi-host cluster reaches
  a Ready state with pods schedulable across hosts.

**Rationale**: mkind operates at the infrastructure layer where
failures are opaque and debugging is expensive. Automated tests
are the primary mechanism for ensuring correctness across host
boundaries.

### IV. Kubernetes API Conventions

All user-facing configuration and CLI interfaces MUST follow
Kubernetes API conventions.

- Cluster configuration MUST use typed, versioned API objects
  (e.g., `apiVersion: mkind.io/v1alpha1`, `kind: Cluster`).
- CLI MUST use cobra with subcommand structure following kubectl
  patterns (e.g., `mkind create cluster`, `mkind get nodes`).
- Structured values MUST live in config files, not CLI flags.
  Flags are reserved for overrides and simple scalar values.
- Output MUST support both human-readable and JSON formats.
- Error messages MUST be written to stderr; data output to stdout.

**Rationale**: mkind targets Kubernetes practitioners. Familiar
conventions reduce learning curve and enable integration with
existing tooling (shell scripts, CI pipelines, kubectl plugins).

### V. Hermetic & Graceful Degradation

Operations MUST be reproducible and self-contained. Partial
failures MUST result in useful diagnostic state, not silent
corruption.

- mkind MUST NOT depend on external services for core cluster
  creation (no cloud APIs, no SaaS dependencies in the critical
  path).
- All cluster state MUST be recoverable from the running
  containers and the cluster config file. No external state stores.
- If a node fails to join, mkind MUST report the failure with
  actionable diagnostics (node IP, kubeadm output, route status)
  and continue bootstrapping remaining nodes where possible.
- Container images required for cluster creation MUST be
  pre-pullable or bundled. Operations MUST NOT fail silently due
  to registry unavailability.
- IP subnet allocation MUST be deterministic given the same config
  input, enabling reproducible cluster topologies.

**Rationale**: mkind will be used in CI/CD, bare-metal labs, and
air-gapped environments. Hermetic design and clear failure modes
are essential for debuggability in these contexts.

## Technical Constraints

- **Language**: Go (aligned with KinD and the Kubernetes ecosystem).
- **License**: Apache 2.0 (aligned with KinD and Kubernetes).
- **CLI framework**: cobra.
- **Bootstrap mechanism**: kubeadm via KinD node images.
- **MVP networking**: kindnetd (unmodified) over Docker bridge with
  per-host /24 subnets from a /16 node subnet pool, connected by
  static IP routes on the host network.
- **Cross-host communication**: SSH for the MVP agent protocol.
  gRPC is a future option but MUST NOT be introduced until SSH
  proves insufficient.
- **Container runtime**: Docker is the only supported runtime for
  the MVP. Podman/nerdctl support is a future consideration.
- **Minimum host requirements**: Linux hosts with Docker installed,
  IP forwarding enabled, and direct L2/L3 reachability between
  hosts (same LAN or routed network).

## Development Workflow

- All changes MUST be submitted via pull requests with at least
  one approval before merge.
- Every PR MUST include or reference tests that cover the changed
  behavior.
- Commits MUST follow conventional commit format
  (e.g., `feat:`, `fix:`, `docs:`, `test:`, `refactor:`).
- CI MUST pass (lint, unit tests, integration tests) before merge.
- Breaking changes to the cluster config schema MUST increment the
  API version (e.g., `v1alpha1` to `v1alpha2` or `v1beta1`).
- Feature branches MUST be named descriptively
  (e.g., `feat/multi-host-bootstrap`, `fix/subnet-collision`).
- Documentation MUST be updated in the same PR as the code change
  it describes.

## Governance

This constitution is the authoritative source for mkind's
development principles and constraints. All pull requests,
code reviews, and architectural decisions MUST be evaluated
against these principles.

- **Amendments** require: (1) a written proposal describing the
  change and its rationale, (2) review and approval, and
  (3) a migration plan if existing code or workflows are affected.
- **Versioning** follows semantic versioning:
  - MAJOR: Principle removed, redefined, or made backward
    incompatible.
  - MINOR: New principle or section added, or existing guidance
    materially expanded.
  - PATCH: Clarifications, wording fixes, non-semantic refinements.
- **Compliance review**: At minimum, each PR review MUST include a
  check that the change does not violate any active principle.
  Complexity introduced by a change MUST be justified in the PR
  description if it appears to conflict with Principle II
  (Simplicity & YAGNI).
- **Deferred principles**: Principles marked with `TODO` are
  acknowledged gaps. They MUST be resolved before the project
  reaches beta maturity. Interim constraints stated alongside the
  TODO are binding until the principle is finalized.
- **Runtime guidance**: Refer to `docs/PROJECT_GOAL_MVP.md` and
  `docs/PROJECT_GOAL_LONGTERM.md` for operational context and
  milestone tracking.

**Version**: 1.0.0 | **Ratified**: 2026-06-30 | **Last Amended**: 2026-06-30
