# Implementation Plan: Progressive Sync Controller

**Branch**: `001-progressive-sync-controller` | **Date**: 2026-09-15 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-progressive-sync-controller/spec.md`

## Summary

Build a standalone Kubernetes controller that orchestrates the progressive (staged, health-gated)
rollout of Argo CD `Application` resources — extracting the RollingSync behavior currently embedded
in the ApplicationSet controller into its own controller and CRD. A `ProgressiveSync` custom
resource selects a set of Applications via a selector composed of a native label selector
**`selector.application`** (Application labels), an optional native label selector
**`selector.namespace`** (which namespaces to coordinate across — defaulting to the ProgressiveSync's
own namespace), and an optional **`selector.installationID`** scoping field. It groups the selected Applications into ordered steps and
triggers each step to sync only after the prior step reaches a success condition (Synced + Healthy),
halting on failure.
Tenancy is enforced by requiring every governed Application to carry the applicable Argo CD
`argocd.argoproj.io/installation-id` — from `spec.selector.installationID` when set, otherwise the
controller's configured bound id. The first design phase (this plan)
focuses on the **CRD `spec` and `status`** shape; deployment manifests, RBAC, and reconciler
implementation follow in later phases/tasks.

## Technical Context

**Language/Version**: Go 1.27 (latest stable toolchain)

**Primary Dependencies**: `sigs.k8s.io/controller-runtime`, `k8s.io/apimachinery`, Kubebuilder-style
scaffolding, `controller-gen` (CRD/deepcopy/RBAC generation); Argo CD `Application` types from
`github.com/argoproj/argo-cd/v3/pkg/apis/application/v1alpha1` (typed or unstructured — see research).

**Reference implementation**: The RollingSync logic being extracted lives in the upstream Argo CD
repo (local checkout `/Users/agaudreault/git/oss/agaudreault/argoproj/argo-cd`, module
`github.com/argoproj/argo-cd/v3`) under `applicationset/progressivesync/progressive_sync.go` (the
`Manager` type) and is wired into `applicationset/controllers/applicationset_controller.go`. See
research.md Decision 2 for the function-to-task mapping.

**Storage**: Kubernetes API server / etcd (CRD-backed state; no external datastore). Status and
rollout history persisted on the `ProgressiveSync` resource `status`.

**Testing**: `go test` for unit tests; `envtest` (controller-runtime) for integration/reconciler
tests; optional e2e against a kind cluster with Argo CD installed.

**Target Platform**: Kubernetes (Linux), runnable in the Argo CD namespace or its own namespace with
multi-namespace Application discovery.

**Project Type**: Kubernetes controller (single Go module, Argo CD-style repository layout).

**Performance Goals**: Reconcile latency dominated by Application sync/health convergence, not
controller overhead; controller should comfortably govern hundreds of Applications per
`ProgressiveSync` with steady-state reconcile CPU negligible.

**Constraints**: Least-privilege RBAC; idempotent, fail-safe reconciliation; namespace scope and
bound installation id are configuration-driven; no modification of Application `spec` desired state
(controller only triggers/gates syncs). **Privilege boundary**: cross-namespace selection
(`spec.selector.namespace`) is allowed only for ProgressiveSyncs in the controller's own namespace;
elsewhere it is restricted to its own namespace and specifying a namespace selector is a
configuration error.

**Scale/Scope**: v1 targets a single Argo CD installation per controller; multiple `ProgressiveSync`
resources; hundreds of governed Applications across multiple watched namespaces.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluated against `.specify/memory/constitution.md` v1.0.0:

| Principle | Gate | Status |
|-----------|------|--------|
| I. Upstream Alignment & Argo CD Conventions | CRD uses `argoproj.io`-style grouping; RollingSync semantics preserved or divergence documented; standard repo layout | PASS — design mirrors ApplicationSet RollingSync; divergences recorded in research.md |
| II. Kubernetes-Native API Design | Versioned CRD, selector-based Application integration, `status` subresource with `conditions` | PASS — `v1alpha1`, `spec.selector`, `status.conditions` (this phase's focus) |
| III. Test-First (NON-NEGOTIABLE) | Unit + envtest integration coverage planned before impl | PASS — testing strategy defined; enforced at tasks/impl phase |
| IV. Declarative Deployment & Least-Privilege RBAC | Full manifests (CRD, Deployment, SA, RBAC, ConfigMap); both topologies; config-driven scope; cross-namespace selection is a privilege reserved to the controller namespace | PASS (planned) — manifest/RBAC design deferred to later phase; privilege boundary strengthens least-privilege posture |
| V. Observability & Operational Safety | Structured logs, metrics, Events; idempotent; fail-safe halt | PASS — status/conditions + fail-safe halting designed in; metrics/events in impl phase |

**Result**: No violations. Complexity Tracking section left empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-progressive-sync-controller/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output (CRD spec + status model — focus)
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── progressivesync.argoproj.io_progressivesyncs.yaml   # CRD schema contract
└── tasks.md             # /speckit-tasks output (NOT created here)
```

### Source Code (repository root)

Argo CD-style / Kubebuilder layout for a single Go module:

```text
api/
└── v1alpha1/
    ├── progressivesync_types.go     # ProgressiveSync spec + status Go types
    ├── groupversion_info.go
    └── zz_generated.deepcopy.go     # generated

cmd/
└── main.go                          # manager entrypoint, flags/config

internal/
└── controller/
    ├── progressivesync_controller.go
    ├── selection.go                 # selector + installation-id tenancy filtering
    ├── steps.go                     # step grouping + progression logic
    └── status.go                    # status/condition computation

config/                             # kustomize-style manifests (Argo CD-aligned)
├── crd/
├── rbac/
├── manager/
└── configmap/

manifests/                          # aggregated installable manifests (install.yaml)

test/
├── e2e/
└── integration/                    # envtest-based reconciler tests
```

**Structure Decision**: Single Go module using controller-runtime/Kubebuilder conventions, with an
Argo CD-aligned `config/` + `manifests/` layout. API types live under `api/v1alpha1`; reconciler and
selection/step/status logic under `internal/controller`. This phase delivers only the design of the
`api/v1alpha1` `ProgressiveSync` `spec`/`status` (captured in data-model.md and the CRD contract);
code generation and controller implementation are later phases.

## Complexity Tracking

> No constitution violations — section intentionally empty.
