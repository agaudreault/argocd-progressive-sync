# Phase 0 Research: Progressive Sync Controller

This document resolves the unknowns needed to design the `ProgressiveSync` CRD `spec`/`status`
(this phase's focus) and to frame the controller implementation.

## Decision 1: API group, version, and Kind

- **Decision**: `apiVersion: argoproj.io/v1alpha1`, `kind: ProgressiveSync`, resource
  `progressivesyncs` (short name `ps`). Start at `v1alpha1`.
- **Rationale**: The constitution mandates Argo CD-style API grouping (Principle I). Progressive sync
  is a core Argo CD concept extracted from ApplicationSet, so sharing the `argoproj.io` group keeps
  it a first-class Argo CD component and eases a future upstream contribution path. `v1alpha1`
  signals evolving API (Principle II: breaking changes require a new version).
- **Alternatives considered**:
  - Dedicated group (e.g., `progressivesync.argoproj.io`, as GitOps Promoter uses
    `promoter.argoproj.io`) — cleaner isolation and no CRD ownership overlap with Argo CD, but weaker
    "first-class Argo CD component" signal. **Recorded as the fallback** if sharing `argoproj.io`
    causes packaging/ownership friction.
  - Reusing an existing Kind — rejected; a new resource is required.

## Decision 2: Reuse of ApplicationSet RollingSync semantics

- **Decision**: Model the strategy on ApplicationSet's `RollingSync`: an ordered list of **steps**,
  each selecting a subset of the governed Applications via label `matchExpressions`, with an optional
  `maxUpdate` (absolute or percentage) controlling concurrency within a step. A step must reach the
  success condition (all its Applications **Synced + Healthy**) before the next step is triggered; a
  step that fails to converge **halts** progression.
- **Rationale**: Principle I requires preserving compatible semantics or documenting divergence.
  Operators already understand RollingSync, minimizing behavioral surprise.
- **Reference implementation (source of truth for behavior)**: Local Argo CD checkout at
  `/Users/agaudreault/git/oss/agaudreault/argoproj/argo-cd` (module `github.com/argoproj/argo-cd/v3`).
  The RollingSync logic is a self-contained `Manager` in
  `applicationset/progressivesync/progressive_sync.go`, wired into
  `applicationset/controllers/applicationset_controller.go`. Function-to-task mapping:
  - `PerformProgressiveSyncs` → reconcile driver (our T020) — top-level per-cycle orchestration.
  - `buildAppDependencyList` / `labelMatchedExpression` / `getAppStep` → step grouping (our T017).
  - `getAppsToSync` → which Apps in the current step to trigger; `SyncDesiredApplications` → sync
    triggering (our T019).
  - `UpdateApplicationSetApplicationStatus` / `UpdateApplicationSetApplicationStatusProgress` →
    per-Application status + Synced/Healthy success evaluation (our T018/T021); note upstream uses
    `ProgressiveSyncHealthy` and gates on `SyncStatusCodeSynced` + `HealthStatusHealthy`.
  - `ensureApplicationsReconciled` / `checkAllApplicationsReconciled` / `needsReconcile` /
    `addRefreshAnnotationToApplications` / `GetLatestWaitingTransitionTimeOfAppset` → the refresh
    grace-period + "apps reconciled by Argo before proceeding" logic our supersede behavior mirrors
    (our T049, FR-016; upstream `RefreshGracePeriodSeconds`).
  - `IsRollingSyncStrategy` / `RollingSyncStrategyEnabled` / `IsStepsEmpty` → strategy/validation
    guards (our T008 types + validation).
- **Key divergence to preserve deliberately**: upstream operates on ApplicationSet-*generated*
  Applications; our controller *selects pre-existing* Applications (no ownership) and adds
  installation-id tenancy filtering (Decision 3) — the grouping/success/grace-period mechanics are
  reused, but selection and ownership differ.
- **Divergences (documented per Principle I)**:
  1. **Standalone selection**: ApplicationSet owns/generates its Applications; the standalone
     controller instead **selects pre-existing** Applications via `spec.selector` (it does not own or
     generate them). It triggers syncs and gates progression; it never mutates Application desired
     state (`spec`).
  2. **Tenancy validation** via installation id (Decision 3) — new, not present in ApplicationSet.
  3. Explicit `status` conditions and rollout history on the CR (Decision 4).

## Decision 3: Tenancy validation via Argo CD installation id

- **Decision**: The controller is bound to a default Argo CD installation id via configuration, and
  a `ProgressiveSync` may optionally override it per-resource via `spec.selector.installationID`.
  During selection the controller reads the `argocd.argoproj.io/installation-id` annotation on each
  candidate Application and includes only those whose value matches the applicable id
  (`spec.selector.installationID` when set, else the controller's configured bound id). Applications
  missing the annotation or with a mismatched value are excluded and surfaced in status. If the
  applicable id cannot be determined, it fails safe and governs nothing.
- **Rationale**: Confirmed via Argo CD resource-tracking docs and `util/argo/resource_tracking.go`:
  when `installationID` is set in `argocd-cm`, Argo CD stamps managed resources with the
  `argocd.argoproj.io/installation-id` annotation to disambiguate multiple instances on one cluster.
  This is the correct signal to ensure a `ProgressiveSync` only governs one instance's Applications
  (spec FR-017–FR-020, User Story 5). Fail-safe matches Principle V.
- **Open item for later phase (not blocking CRD design)**: whether the bound id is supplied purely
  via controller config/flag, or optionally discovered from a co-located `argocd-cm`. Default: config
  flag `--installation-id` (and/or ConfigMap key), empty means "no installation id" → only match
  Applications that also have no installation-id annotation.
- **Alternatives considered**: label-based `app.kubernetes.io/instance` — rejected; truncated and not
  installation-scoped. Namespace-only isolation — rejected; insufficient when instances share
  namespaces.

## Decision 4: Status shape — phases, per-application status, conditions, history

- **Decision**: `status` carries: `observedGeneration`, a top-level `phase`
  (`Pending|Progressing|Healthy|Error` — no `Paused`), `currentStepIndex`/`currentStepName`, a list of
  `applicationStatuses` (application ref, assigned step, sync + health status, message), standard
  `conditions[]` (`type/status/reason/message/lastTransitionTime/observedGeneration`), timestamps
  (`startedAt`/`finishedAt`), and a bounded `history[]` of past rollouts with completion state.
- **Rationale**: Principle II requires a `status` subresource with standard `conditions`; the UI
  feature (002) consumes exactly these fields (current operation summary, selected apps, conditions,
  history). Bounded history keeps the object size safe (default keep last N, e.g., 10).
- **Alternatives considered**: separate history CRD/records — deferred; on-CR bounded history is
  simplest for v1 and satisfies the UI needs. Revisit if retention depth demands it.

## Decision 5: Success condition definition

- **Decision**: Default success condition for a step = all Applications in the step have
  `status.sync.status == Synced` **and** `status.health.status == Healthy`. v1 hardcodes
  Synced+Healthy. If a step is not yet Synced+Healthy, the rollout stays `Progressing` and waits on
  that step **indefinitely** — there is no separate `Paused` phase, no step timeout, and no rollback.
- **Rationale**: Matches the clarified spec (FR-004) and ApplicationSet behavior; waiting rather than
  timing out is the fail-safe default (Principle V) and keeps the state machine simple.

## Decision 6: Namespace scoping / multi-namespace discovery

- **Decision**: Two layers of namespace scoping:
  1. **Controller-level (RBAC bound)**: the set of namespaces the controller is *allowed* to watch is
     configuration-driven (flag/ConfigMap, e.g., `--application-namespaces`), defaulting to the
     controller's own namespace. Supports Argo CD-namespace and dedicated-namespace installs.
  2. **Per-resource (`spec.selector.namespace`)**: each ProgressiveSync chooses which namespaces
     (within the allowed set) to coordinate Applications across, using a native Kubernetes
     `metav1.LabelSelector` over `Namespace` objects. Omitted ⇒ only the ProgressiveSync's own
     namespace. Namespaces resolved outside the controller's allowed set are ignored and surfaced.
- **Security constraint (privilege boundary)**: Cross-namespace selection is a privileged operation
  restricted by where the `ProgressiveSync` lives (modeled on Argo CD's apps-in-any-namespace trust
  model):
  - A `ProgressiveSync` created in the **controller's own namespace** (the
    `progressive-sync-controller` namespace) MAY use `spec.selector.namespace` to govern Applications
    in other namespaces.
  - A `ProgressiveSync` created in **any other namespace** is restricted to **its own namespace
    only**. Specifying `spec.selector.namespace` in that case is a **configuration error**: the
    controller MUST NOT govern anything for that resource and MUST surface an `Error` phase with a
    `ConfigurationError` condition (fail-safe), rather than silently ignoring the selector.
  This prevents a tenant in a delegated namespace from reaching Applications outside their namespace.
- **Rationale**: Principle IV (config-driven scope, both topologies, least privilege — RBAC scoped to
  the allowed namespaces) and Principle V (fail-safe). The per-resource selector lets one controller
  coordinate multi-namespace rollouts without widening RBAC beyond the configured bound, while the
  privilege boundary ensures only the trusted controller namespace can author cross-namespace
  rollouts. Watching across namespaces uses controller-runtime cache/`MultiNamespacedCacheBuilder`-
  style scoping (Kubebuilder-native).
- **Alternatives considered**: cluster-wide watch — rejected (violates least-privilege). Name-glob
  patterns (e.g. `my-app-*`) — rejected in favor of native label selection only, which is standard
  Kubernetes and Kubebuilder-idiomatic (no custom glob matching).

## Decision 7: Argo CD Application type usage

- **Decision**: Read Applications via the typed Argo CD `Application` API where practical; otherwise
  unstructured to avoid a heavy dependency. Only read `metadata` (labels/annotations) and
  `status.sync`/`status.health`; trigger sync by setting `operation` (the same mechanism Argo CD/CLI
  uses) rather than editing `spec`.
- **Rationale**: Principle I/II; avoids mutating desired state (spec assumption). Final import choice
  is an implementation detail resolved at coding time; does not affect the CRD contract.

## Summary of resolved unknowns

All Technical Context items are resolved; no remaining NEEDS CLARIFICATION for the CRD design phase.
Deferred (non-blocking) items are explicitly scoped to later phases: exact installation-id discovery
source, configurable success conditions, and full RBAC/manifest layout.
