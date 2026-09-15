# Phase 1 Data Model: ProgressiveSync CRD

This is the focus deliverable for the first design phase: the `ProgressiveSync` CRD `spec` and
`status`. The machine-readable schema lives in
[`contracts/progressivesync.argoproj.io_progressivesyncs.yaml`](./contracts/progressivesync.argoproj.io_progressivesyncs.yaml).

- **Group/Version/Kind**: `argoproj.io/v1alpha1`, `ProgressiveSync` (resource `progressivesyncs`,
  short name `ps`), namespaced.
- **Subresources**: `status` enabled.

## Entity: ProgressiveSync

Top-level object. `spec` is user-authored intent; `status` is controller-reported observed state.

### spec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `selector` | `ApplicationSelector` | yes | Selects the Argo CD Applications this ProgressiveSync governs (app labels, namespace scope, and optional installation-id scoping). |
| `strategy` | `Strategy` | yes | How the governed Applications are rolled out. |

#### ApplicationSelector

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `application` | `metav1.LabelSelector` | no | Native Kubernetes label selector over Application labels (`matchLabels`/`matchExpressions`). Empty = all Applications in the resolved namespace scope. |
| `namespace` | `metav1.LabelSelector` | no | Native Kubernetes label selector over `Namespace` objects, choosing which namespaces to search for Applications. When omitted, defaults to **only the ProgressiveSync's own namespace**. Enables coordinating Applications across multiple namespaces via namespace labels. |
| `installationID` | `string` | no | Optionally scopes selection to a specific Argo CD installation id (`argocd.argoproj.io/installation-id`). When empty, the controller's configured bound installation id applies. Used for tenancy validation (FR-017–FR-020). |

Both `application` and `namespace` are standard `metav1.LabelSelector` values (label-based only; no
name globbing). All namespaces resolved by `namespace` MUST fall within the controller's configured
allowed namespace scope (RBAC-bounded, see research Decision 6); namespaces outside that scope are
ignored and surfaced.

#### Strategy

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `string` enum: `RollingSync` | yes | Rollout strategy type. Only `RollingSync` in v1. |
| `rollingSync` | `RollingSyncStrategy` | when `type=RollingSync` | Ordered-step rollout configuration. |

#### RollingSyncStrategy

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `steps` | `[]RollingSyncStep` | yes (min 1) | Ordered list of rollout steps; executed in array order. |

#### RollingSyncStep

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | no | Human-friendly step name (surfaced in status/UI). |
| `matchExpressions` | `[]ApplicationMatchExpression` | no | Label expressions selecting which governed Applications belong to this step. Empty = all remaining governed Applications. |
| `maxUpdate` | `intstr.IntOrString` | no | Max Applications synced concurrently within the step (absolute int or percentage string, e.g. `"25%"`). Default: all at once. |

#### ApplicationMatchExpression

Mirrors label selector requirement semantics (compatible with ApplicationSet RollingSync).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key` | `string` | yes | Application label key. |
| `operator` | `string` (`In`/`NotIn`/`Exists`/`DoesNotExist`) | yes | Set-based operator. |
| `values` | `[]string` | conditional | Required for `In`/`NotIn`. |

### status

| Field | Type | Description |
|-------|------|-------------|
| `observedGeneration` | `int64` | `metadata.generation` last reconciled. |
| `phase` | `string` enum: `Pending`/`Progressing`/`Healthy`/`Error` | Overall rollout state. A rollout waiting on an unhealthy step stays `Progressing` (there is no `Paused` phase, no timeout, no rollback). |
| `message` | `string` | Human-readable summary of the current phase. |
| `currentStepIndex` | `int` | Zero-based index of the active step (`-1` when none). |
| `currentStepName` | `string` | Name of the active step (if named). |
| `startedAt` | `metav1.Time` | When the current rollout started. |
| `finishedAt` | `metav1.Time` | When the current rollout completed (success or terminal halt). |
| `applicationStatuses` | `[]ApplicationStatus` | Per-Application observed state within the rollout. |
| `conditions` | `[]metav1.Condition` | Standard conditions (see below). |
| `history` | `[]RolloutRecord` | Bounded list (default last 10) of past rollouts. |

#### ApplicationStatus

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | Application name. |
| `namespace` | `string` | Application namespace. |
| `step` | `int` | Step index this Application is assigned to. |
| `syncStatus` | `string` | Observed Argo CD sync status (e.g. `Synced`/`OutOfSync`). |
| `healthStatus` | `string` | Observed Argo CD health status (e.g. `Healthy`/`Degraded`/`Progressing`). |
| `status` | `string` enum: `Pending`/`Progressing`/`Completed`/`Excluded`/`Error` | Rollout-relative status. `Excluded` = filtered out by tenancy validation. |
| `message` | `string` | Reason detail (e.g. installation-id mismatch, missing annotation). |

#### RolloutRecord (history entry)

| Field | Type | Description |
|-------|------|-------------|
| `startedAt` | `metav1.Time` | Rollout start. |
| `finishedAt` | `metav1.Time` | Rollout end. |
| `result` | `string` enum: `Completed`/`Superseded` | Completion state — `Completed` (all steps succeeded) or `Superseded` (a new change replaced the in-flight rollout). No `Aborted`/`Failed`. |
| `message` | `string` | Summary / failure reason. |
| `revisionHash` | `string` | Optional identifier of the change that drove the rollout. |

### Condition types (values of `conditions[].type`)

| Type | Meaning |
|------|---------|
| `Ready` | Controller successfully reconciling this resource. |
| `Progressing` | A rollout is advancing through steps, or waiting indefinitely on a step whose Applications are not yet Synced + Healthy. |
| `Error` | Reconciliation or validation error (e.g., unknown bound installation id). |
| `ConfigurationError` | The resource is misconfigured — e.g., a ProgressiveSync outside the controller namespace specified `spec.selector.namespace`. Controller governs nothing (fail-safe). |
| `TenancyValidated` | All governed Applications matched the bound installation id (False lists exclusions in message). |

## Validation Rules (from spec requirements)

- `spec.selector` required; a ProgressiveSync with a selector matching zero Applications reports
  `phase: Healthy` (idle), never an error (FR-014, SC-007).
- `spec.selector.namespace` omitted ⇒ namespace scope is exactly the ProgressiveSync's own
  namespace. When present, it may resolve to multiple namespaces via native label selection, each
  constrained to the controller's configured allowed namespace scope (FR-009/FR-010).
- **Privilege boundary (security)**: `spec.selector.namespace` is permitted ONLY when the
  ProgressiveSync resides in the controller's own namespace. If a ProgressiveSync in any other
  namespace specifies `spec.selector.namespace`, it is a configuration error: the controller MUST set
  `phase: Error` with a `ConfigurationError` condition and govern no Applications (fail-safe). A
  ProgressiveSync outside the controller namespace is always restricted to its own namespace.
- `strategy.type` must be `RollingSync`; `rollingSync.steps` must have ≥ 1 step.
- `maxUpdate` percentage strings must parse as `0–100%`; absolute values must be ≥ 0.
- Every Application in `applicationStatuses` with `status: Completed` MUST belong to the applicable
  installation id (`spec.selector.installationID` if set, else the controller's configured id)
  (FR-020); mismatches/missing → `status: Excluded` with message (FR-017–FR-018).
- Progression to step `n+1` requires all non-excluded Applications in step `n` to be `Synced` AND
  `Healthy` (Decision 5); until then the rollout stays `Progressing` on step `n` indefinitely — no
  `Paused` phase, no timeout, no rollback (FR-003/FR-004).
- Reconciliation is idempotent; identical inputs yield identical status (FR-012).

## State Transitions (phase)

```text
Pending ──(apps selected, first step triggered)──────────▶ Progressing
Progressing ──(step success, more steps)─────────────────▶ Progressing (next step)
Progressing ──(step not yet Synced+Healthy)──────────────▶ Progressing (waits on step, indefinitely)
Progressing ──(all steps succeed)────────────────────────▶ Healthy
any ──(validation/reconcile error)───────────────────────▶ Error
Healthy/Error ──(new change: grace period + Argo refresh)▶ Pending (record prior rollout in history)
```

## Relationships

- `ProgressiveSync` **selects** many external Argo CD `Application`s (by label; filtered by
  installation id). It does not own them (no owner references, no `spec` mutation).
- Consumed by feature 002 (UI): `status.phase`/`message`/`currentStep*` → current-operation summary;
  `applicationStatuses` → selected-apps list + Argo CD navigation; `conditions` → diagnostics;
  `history` → sync history.
