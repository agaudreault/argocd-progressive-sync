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
- **Divergences (documented per Principle I)**:
  1. **Standalone selection**: ApplicationSet owns/generates its Applications; the standalone
     controller instead **selects pre-existing** Applications via `spec.selector` (it does not own or
     generate them). It triggers syncs and gates progression; it never mutates Application desired
     state (`spec`).
  2. **Tenancy validation** via installation id (Decision 3) — new, not present in ApplicationSet.
  3. Explicit `status` conditions and rollout history on the CR (Decision 4).

## Decision 3: Tenancy validation via Argo CD installation id

- **Decision**: The controller is bound to exactly one Argo CD installation id via configuration.
  During selection it reads the `argocd.argoproj.io/installation-id` annotation on each candidate
  Application and includes only those whose value matches the bound id. Applications missing the
  annotation or with a mismatched value are excluded and surfaced in status. If the controller's
  bound id cannot be determined, it fails safe and governs nothing.
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
  (`Pending|Progressing|Paused|Healthy|Error`), `currentStepIndex`/`currentStepName`, a list of
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
  `status.sync.status == Synced` **and** `status.health.status == Healthy`. Configurable in a later
  iteration; v1 hardcodes Synced+Healthy with an optional per-step timeout that, when exceeded, sets
  `Paused` with a reason (fail-safe; no auto-advance).
- **Rationale**: Matches spec assumptions and ApplicationSet behavior; Principle V fail-safe.

## Decision 6: Namespace scoping / multi-namespace discovery

- **Decision**: Watched Application namespaces are configuration-driven (flag/ConfigMap, e.g.,
  `--application-namespaces`), defaulting to the controller's own namespace. Supports both Argo CD
  namespace install and dedicated-namespace multi-namespace install.
- **Rationale**: Principle IV (config-driven scope, both topologies, least privilege — RBAC scoped to
  the watched namespaces). Detailed RBAC/manifest design is a later phase; no impact on CRD shape.

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
