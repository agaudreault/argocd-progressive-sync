<!--
Sync Impact Report
==================
Version change: 1.0.0 → 1.1.0
Bump rationale: Materially re-scoped Principle I to prefer Kubebuilder/controller-runtime
implementation patterns over Argo CD-specific/legacy ones, while keeping Argo CD conventions
authoritative for the API surface and user-facing behavior (MINOR — expanded/clarified guidance,
no principle removed).

Modified principles:
  - I. Upstream Alignment & Argo CD Conventions → I. Argo CD API Alignment, Kubebuilder Implementation
Added principles: none
Added sections: none
Removed sections: none

Prior baseline (v1.0.0) principles:
  - I. Upstream Alignment & Argo CD Conventions
  - II. Kubernetes-Native API Design
  - III. Test-First (NON-NEGOTIABLE)
  - IV. Declarative Deployment & Least-Privilege RBAC
  - V. Observability & Operational Safety

Templates & guidance requiring review for alignment:
  - .specify/templates/plan-template.md (Constitution Check gate) — verify referenced sections
  - .specify/templates/spec-template.md — no change required
  - .specify/templates/tasks-template.md — no change required
  - specs/001-progressive-sync-controller/plan.md — Structure Decision already Kubebuilder-aligned

Follow-up TODOs: none
-->

# Argo CD Progressive Sync Controller Constitution

## Core Principles

### I. Argo CD API Alignment, Kubebuilder Implementation

Argo CD conventions are authoritative for the **API surface and user-facing behavior**: CRD
`argoproj.io`-style API grouping, label/annotation conventions, `Application` integration, and
installation-id tenancy. The progressive sync behavior extracted from the ApplicationSet controller
MUST preserve compatible semantics unless a divergence is explicitly documented and justified.

For the **controller implementation** — project structure, scaffolding, reconcile idioms, tooling,
and code generation — the project MUST prefer current Kubebuilder / `controller-runtime` patterns
over Argo CD-specific or legacy patterns. Where an Argo CD-specific approach carries technical debt
or conflicts with modern Kubebuilder practice, the Kubebuilder approach MUST win unless a specific,
documented reason requires otherwise.

Rationale: Argo CD conventions on the API keep the controller a first-class, familiar Argo CD
component with an open upstream-contribution path; Kubebuilder is newer and better maintained, so
building on it for implementation avoids inheriting Argo CD's older controller patterns and
technical debt.

### II. Kubernetes-Native API Design

The controller MUST expose its behavior through a versioned CRD and reconcile via the
Kubernetes control loop. The API MUST integrate with `Application` resources through a
label/field **selector** rather than hard references where a set of targets is intended. Every
custom resource MUST expose a `status` subresource with standard `conditions` and observed state.
API changes MUST be additive within a version; breaking changes REQUIRE a new API version.

Rationale: Selector-based, status-driven, versioned APIs are the idiomatic Kubernetes contract and
make the controller composable with existing Argo CD Applications and multi-namespace topologies.

### III. Test-First (NON-NEGOTIABLE)

Test-Driven Development is mandatory. For every reconciliation behavior, tests MUST be written
first, MUST fail before implementation, and MUST pass after. Controller logic MUST be covered by
unit tests, and reconcilers MUST have integration tests using envtest (a real API server) or an
equivalent controller-runtime test harness. No reconciliation logic merges without accompanying
tests.

Rationale: Controllers manage live cluster state; untested reconciliation causes real, hard-to-
reverse damage. Red-Green-Refactor is the guardrail.

### IV. Declarative Deployment & Least-Privilege RBAC

The controller MUST ship complete, declarative manifests to deploy itself, including the CRD,
Deployment, `ServiceAccount`, RBAC (`Role`/`ClusterRole` and bindings), and `ConfigMap` for
configuration. RBAC MUST grant the minimum permissions required. The deployment MUST support both
topologies: running inside the Argo CD namespace, and running in its own namespace while
discovering `Application` resources across multiple namespaces. Namespace scope MUST be
configuration-driven, never hard-coded.

Rationale: Operators expect turnkey, auditable manifests; least-privilege and flexible namespace
scoping are baseline security and operability requirements for a cluster controller.

### V. Observability & Operational Safety

The controller MUST emit structured logs, Prometheus-style metrics for reconcile counts, errors,
and durations, and Kubernetes `Events` for significant state transitions. Reconciliation MUST be
idempotent and safe to retry. Progressive sync operations MUST fail safe: on ambiguity or error the
controller MUST halt progression rather than continue rolling out changes.

Rationale: Progressive sync governs production rollouts; visibility and fail-safe behavior are what
make automated rollouts trustworthy.

## Technical Constraints & Standards

- Language: Go, using the current Argo CD-supported Go toolchain version.
- Framework: `controller-runtime` with Kubebuilder scaffolding and project layout for controllers
  and CRDs. Kubebuilder conventions take precedence over Argo CD-specific implementation patterns.
- Code generation: CRD manifests, deepcopy functions, and RBAC MUST be generated (via
  `controller-gen`/Kubebuilder tooling) from typed Go API definitions and kept in sync with source
  (no hand-drift).
- Formatting & linting: code MUST pass `gofmt`/`goimports` and the project linter before merge.
- Manifests: MUST be organized under a Kubebuilder-style layout (e.g., `config/` with an aggregated
  installable path), installable via a single apply path; Argo CD packaging expectations are met at
  the distribution layer rather than by adopting legacy Argo CD manifest patterns.
- Multi-namespace discovery MUST be governed by explicit configuration (e.g., ConfigMap and/or
  flags), defaulting to safe, restricted scope.

## Development Workflow & Quality Gates

- All changes land via pull request; direct pushes to the main branch are prohibited.
- CI MUST run build, unit tests, integration (envtest) tests, and lint; all MUST pass to merge.
- Generated artifacts (CRDs, deepcopy, RBAC) MUST be regenerated and committed when API types
  change; a CI check SHOULD verify no drift.
- Any deviation from Argo CD **API** conventions, or any use of an Argo CD-specific implementation
  pattern in preference to the modern Kubebuilder approach, or added complexity, MUST be justified in
  the PR description and reviewed against this constitution.
- Every new reconciliation feature MUST include documentation of its CRD fields and behavior.

## Governance

This constitution supersedes other practices for the project. Amendments MUST be proposed via pull
request, documented in the Sync Impact Report, reviewed, and approved by a project maintainer.

Versioning of this constitution follows semantic versioning:
- MAJOR: backward-incompatible governance changes or principle removals/redefinitions.
- MINOR: a new principle or materially expanded section.
- PATCH: clarifications and non-semantic refinements.

All pull requests and reviews MUST verify compliance with these principles. Complexity that
violates a principle MUST be justified or removed. Runtime development guidance lives alongside the
codebase (e.g., contributor docs and agent guidance files) and MUST remain consistent with this
constitution.

**Version**: 1.1.0 | **Ratified**: 2026-09-15 | **Last Amended**: 2026-09-15
