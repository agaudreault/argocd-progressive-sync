# Quickstart & Validation: Progressive Sync Controller

This guide validates the `ProgressiveSync` CRD design end-to-end. It focuses on proving the `spec`
and `status` contract behaves as specified. It references the design artifacts rather than
duplicating them:
[data-model.md](./data-model.md) ·
[contracts/progressivesync.argoproj.io_progressivesyncs.yaml](./contracts/progressivesync.argoproj.io_progressivesyncs.yaml).

## Prerequisites

- A Kubernetes cluster (e.g., `kind`) with Argo CD installed.
- `installationID` set in `argocd-cm` (e.g., `argocd.argoproj.io/installation-id: dev-argocd`) so
  managed resources/Applications carry the `argocd.argoproj.io/installation-id` annotation.
- Several Argo CD `Application`s labeled for selection (e.g., `app-group: web`) with an env label used
  for stepping (e.g., `env: dev|staging|prod`).
- The Progressive Sync controller running (Argo CD namespace or dedicated namespace) bound to the
  same installation id (`--installation-id dev-argocd`).

## Setup

1. Apply the CRD:
   ```sh
   kubectl apply -f specs/001-progressive-sync-controller/contracts/progressivesync.argoproj.io_progressivesyncs.yaml
   ```
2. Create a `ProgressiveSync` selecting the labeled Applications with two ordered steps:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ProgressiveSync
   metadata:
     name: web-rollout
     namespace: argocd
   spec:
     selector:
       application:
         matchLabels:
           app-group: web
       namespace:
         matchLabels:
           team: my-app          # omit to scope to this ProgressiveSync's namespace only
     strategy:
       type: RollingSync
       rollingSync:
         steps:
           - name: canary
             matchExpressions:
               - { key: env, operator: In, values: [dev] }
             maxUpdate: 1
           - name: rest
             matchExpressions:
               - { key: env, operator: NotIn, values: [dev] }
   ```

## Validation Scenarios

Each scenario maps to acceptance criteria in [spec.md](./spec.md).

### Scenario A — Ordered, health-gated progression (US1 / FR-003, FR-004)

1. Introduce a change that makes the selected Applications `OutOfSync`.
2. **Expect**: only step `canary` (env=dev) Applications sync first;
   `status.currentStepIndex: 0`, `status.phase: Progressing`.
3. Once the dev Applications are `Synced` + `Healthy`, **expect** step `rest` begins and
   `status.currentStepIndex` advances to `1`.
4. Force a dev Application to stay unhealthy; **expect** `status.phase` stays `Progressing` on step 0
   **indefinitely** (no timeout, no `Paused` phase, no rollback) and no `rest` Applications changed.
5. When all steps succeed, **expect** `status.phase: Healthy` and `finishedAt` set.

### Scenario B — Selector membership (US2 / FR-002, FR-005)

1. Add a new Application labeled `app-group: web`; **expect** it appears in
   `status.applicationStatuses` on next reconcile.
2. Remove the label from an Application; **expect** it drops out of `applicationStatuses`.

### Scenario C — Tenancy validation (US5 / FR-017–FR-020, SC-008)

1. Create an Application matching the selector but with a different / missing
   `argocd.argoproj.io/installation-id`.
2. **Expect**: it is listed with `status: Excluded` and a message explaining the mismatch; it is never
   triggered to sync; the `TenancyValidated` condition reflects the exclusion.

### Scenario F — Multi-namespace scope (FR-009, FR-010)

1. Create matching Applications in namespaces `my-app-1` and `my-app-2`, both labeled `team: my-app`
   (within the controller's allowed scope).
2. With `selector.namespace.matchLabels: {team: my-app}`, **expect** Applications from both
   namespaces appear in `status.applicationStatuses`.
3. Remove `selector.namespace`; **expect** only Applications in the ProgressiveSync's own namespace
   are governed.

### Scenario G — Cross-namespace privilege boundary (security)

1. Create a ProgressiveSync **in the controller namespace** with `selector.namespace`; **expect** it
   governs Applications across the selected namespaces.
2. Create a ProgressiveSync **in a non-controller namespace** with `selector.namespace` set;
   **expect** `phase: Error` with a `ConfigurationError` condition and no Applications governed.
3. Create a ProgressiveSync in a non-controller namespace **without** `selector.namespace`; **expect**
   it governs only Applications in its own namespace.

### Scenario D — Empty selection is not an error (Edge case / SC-007)

1. Point `spec.selector` at labels matching no Applications.
2. **Expect**: `status.phase: Healthy` (idle), no error condition.

### Scenario E — History (supports feature 002)

1. Trigger, complete, then trigger a second rollout.
2. **Expect**: the prior rollout appears in `status.history` with a `result` of
   `Completed` or `Superseded`.

## Automated coverage (per Constitution Principle III)

- **Unit**: selection/tenancy filtering, step grouping, success-condition evaluation, status/phase
  computation, history bounding.
- **Integration (envtest)**: apply a `ProgressiveSync` + fake Applications; assert status transitions
  for Scenarios A–E against a real API server.

## Done (design phase)

- [ ] CRD contract applies cleanly to a cluster (schema valid).
- [ ] Scenarios A–E are expressible against the `spec`/`status` contract without gaps.
- [ ] Reviewers confirm the `status` fields satisfy feature 002 (UI) needs.
