---

description: "Task list for Progressive Sync Controller implementation"
---

# Tasks: Progressive Sync Controller

**Input**: Design documents from `/specs/001-progressive-sync-controller/`

**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

**Tests**: Included — the project constitution (Principle III: Test-First) is NON-NEGOTIABLE, so
unit + envtest integration tests are written before implementation for each story.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4, US5)
- Include exact file paths in descriptions

## Path Conventions

Single Go module, controller-runtime/Kubebuilder layout with Argo CD-aligned manifests (per plan.md):

- API types: `api/v1alpha1/`
- Controller logic: `internal/controller/`
- Manager entrypoint: `cmd/main.go`
- Manifests: `config/` (kustomize) + `manifests/` (aggregated)
- Tests: `internal/controller/*_test.go` (unit), `test/integration/` (envtest), `test/e2e/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Initialize Go module `go.mod` (Go 1.27) at repo root and create the directory skeleton (`api/v1alpha1/`, `internal/controller/`, `cmd/`, `config/{crd,rbac,manager,configmap}`, `manifests/`, `test/{integration,e2e}`) per plan.md
- [ ] T002 Add core dependencies to `go.mod`: `sigs.k8s.io/controller-runtime`, `k8s.io/apimachinery`, `k8s.io/client-go`, and Argo CD `Application` types (`github.com/argoproj/argo-cd/v3/pkg/apis/application/v1alpha1`)
- [ ] T003 [P] Add `Makefile` with `controller-gen` targets (`manifests`, `generate`), `test`, `lint`, and `build` targets
- [ ] T004 [P] Configure linting/formatting (`.golangci.yml`, `gofmt`/`goimports`) and `.gitignore` entries for build artifacts
- [ ] T005 [P] Add `PROJECT` file and `controller-gen` tool dependency (`tools.go` / bin pinning) for CRD/deepcopy/RBAC generation

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core API types, scaffolding, and manager wiring that ALL user stories depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T006 [P] Create `api/v1alpha1/groupversion_info.go` registering group `argoproj.io`, version `v1alpha1`, and scheme builder (per research Decision 1)
- [ ] T007 Define `ProgressiveSync`, `ProgressiveSyncList`, `ProgressiveSyncSpec`, and `ProgressiveSyncStatus` root types in `api/v1alpha1/progressivesync_types.go` with kubebuilder markers (resource `progressivesyncs`, shortname `ps`, status subresource) (data-model.md)
- [ ] T008 Add `spec` sub-types to `api/v1alpha1/progressivesync_types.go`: `ApplicationSelector` (`application`/`namespace` `metav1.LabelSelector`, `installationID`), `Strategy`, `RollingSyncStrategy`, `RollingSyncStep`, `ApplicationMatchExpression` (data-model.md)
- [ ] T009 Add `status` sub-types to `api/v1alpha1/progressivesync_types.go`: `ApplicationStatus`, `RolloutRecord` (result enum `Completed`/`Superseded`), phase enum (`Pending`/`Progressing`/`Healthy`/`Error`), and condition type constants (`Ready`/`Progressing`/`Error`/`ConfigurationError`/`TenancyValidated`) (data-model.md)
- [ ] T010 Run `controller-gen` to produce `api/v1alpha1/zz_generated.deepcopy.go` and the CRD manifest `config/crd/argoproj.io_progressivesyncs.yaml`; reconcile it against `contracts/progressivesync.argoproj.io_progressivesyncs.yaml`
- [ ] T011 Implement controller configuration in `cmd/main.go` and `internal/controller/config.go`: flags `--installation-id`, `--application-namespaces`, `--controller-namespace`, plus ConfigMap binding, manager setup, and `MultiNamespacedCache`-style scoping (research Decision 6)
- [ ] T012 [P] Set up structured logging, metrics registry, and Event recorder scaffolding in `internal/controller/` (Principle V observability)
- [ ] T013 [P] Set up `envtest` harness in `test/integration/suite_test.go` (install the CRD, register scheme, fake Argo CD `Application` CRD) so integration tests can run against a real API server

**Checkpoint**: API types compile, CRD installs, manager boots — user story implementation can begin

---

## Phase 3: User Story 1 - Ordered, health-gated staged rollout (Priority: P1) 🎯 MVP

**Goal**: Given a `ProgressiveSync` with ordered steps, sync each step only after the prior step is Synced + Healthy, halting/waiting indefinitely on a step that never converges, and reporting completion.

**Independent Test**: Apply a `ProgressiveSync` with two steps over fake Applications; make them OutOfSync; assert step 0 syncs first, step 1 only after step 0 is Synced+Healthy, an unhealthy step 0 keeps `phase: Progressing` indefinitely, and all-success yields `phase: Healthy` (quickstart Scenario A).

### Tests for User Story 1 ⚠️

- [ ] T014 [P] [US1] Unit tests for step grouping (assign governed Apps to steps via `matchExpressions`, `maxUpdate` int/percentage) in `internal/controller/steps_test.go`
- [ ] T015 [P] [US1] Unit tests for success-condition evaluation (all step Apps Synced+Healthy → advance; else stay) in `internal/controller/steps_test.go`
- [ ] T016 [P] [US1] Integration (envtest) test for ordered progression + indefinite wait + completion in `test/integration/rollout_test.go` (Scenario A)

### Implementation for User Story 1

- [ ] T017 [US1] Implement step grouping in `internal/controller/steps.go` (assign Applications to steps, resolve `maxUpdate` concurrency)
- [ ] T018 [US1] Implement success-condition + progression logic in `internal/controller/steps.go` (Synced+Healthy gate, advance/wait, no timeout/rollback per FR-003/FR-004)
- [ ] T019 [US1] Implement Application sync triggering in `internal/controller/sync.go` by setting `operation` on the Argo CD `Application` (never mutating `spec`, per research Decision 7)
- [ ] T020 [US1] Implement the core `Reconcile` loop skeleton in `internal/controller/progressivesync_controller.go` (fetch PS, resolve governed set, drive current step, requeue) wiring T017–T019
- [ ] T021 [US1] Implement phase/status computation for the rollout (`phase`, `currentStepIndex/Name`, `startedAt/finishedAt`, `applicationStatuses`) in `internal/controller/status.go`
- [ ] T022 [US1] Implement empty-selection handling → `phase: Healthy` (idle, not error) in `internal/controller/status.go` (FR-014, SC-007)

**Checkpoint**: A `ProgressiveSync` performs ordered, health-gated rollouts end-to-end — MVP demoable

---

## Phase 4: User Story 2 - Selector-based Application membership (Priority: P1)

**Goal**: Define the governed set via `spec.selector.application` label selector; membership updates automatically as Application labels change.

**Independent Test**: Create a `ProgressiveSync` with a label selector; create matching/non-matching Applications; assert only matches are governed and label changes add/remove membership (quickstart Scenario B).

### Tests for User Story 2 ⚠️

- [ ] T023 [P] [US2] Unit tests for label-selector membership resolution (match/no-match, add/remove on label change) in `internal/controller/selection_test.go`
- [ ] T024 [P] [US2] Integration (envtest) test for dynamic membership updates in `test/integration/selection_test.go` (Scenario B)

### Implementation for User Story 2

- [ ] T025 [US2] Implement Application selection by `spec.selector.application` (`metav1.LabelSelector`) in `internal/controller/selection.go`
- [ ] T026 [US2] Add watch/index on Applications and enqueue affected `ProgressiveSync`s on Application create/update/delete in `internal/controller/progressivesync_controller.go` (FR-005)
- [ ] T027 [US2] Ensure deleted/mid-rollout Applications are removed from the governed set and never block progression (edge case) in `internal/controller/selection.go`

**Checkpoint**: Governed set is selector-driven and reacts to Application changes

---

## Phase 5: User Story 5 - Tenancy validation via installation id (Priority: P1)

**Goal**: Only govern Applications whose `argocd.argoproj.io/installation-id` matches the applicable id (`spec.selector.installationID` or the controller's bound id); exclude and surface mismatches/missing; fail safe when the id is unknown.

**Independent Test**: With Applications carrying different/missing installation ids, assert only matching ones are governed, others are `Excluded` with a reason, and `TenancyValidated` reflects exclusions (quickstart Scenario C).

### Tests for User Story 5 ⚠️

- [ ] T028 [P] [US5] Unit tests for installation-id filtering (match / mismatch / missing / unknown bound id → fail-safe) in `internal/controller/selection_test.go`
- [ ] T029 [P] [US5] Integration (envtest) test for exclusion + `TenancyValidated` condition in `test/integration/tenancy_test.go` (Scenario C)

### Implementation for User Story 5

- [ ] T030 [US5] Implement installation-id resolution precedence (`spec.selector.installationID` else controller bound id; empty→only match Apps with no annotation) in `internal/controller/selection.go` (research Decision 3)
- [ ] T031 [US5] Implement tenancy filtering during selection: exclude mismatched/missing-annotation Applications, mark `status: Excluded` with message (FR-017/FR-018) in `internal/controller/selection.go`
- [ ] T032 [US5] Implement fail-safe when applicable installation id cannot be determined (govern nothing, set `Error`) (FR-019) in `internal/controller/selection.go`
- [ ] T033 [US5] Populate the `TenancyValidated` condition and enforce single-installation guarantee (FR-020) in `internal/controller/status.go`

**Checkpoint**: Selection is tenancy-safe across multi-instance environments

---

## Phase 6: User Story 3 - Flexible multi-namespace deployment + privilege boundary (Priority: P2)

**Goal**: Ship deployable manifests for both topologies (Argo CD namespace / dedicated namespace); support multi-namespace discovery via `spec.selector.namespace`; enforce the cross-namespace privilege boundary with least-privilege RBAC.

**Independent Test**: Apply manifests in both topologies; assert multi-namespace selection works from the controller namespace, and a `ProgressiveSync` in another namespace specifying `selector.namespace` is a `ConfigurationError` governing nothing (quickstart Scenarios F, G).

### Tests for User Story 3 ⚠️

- [ ] T034 [P] [US3] Unit tests for namespace-scope resolution (`selector.namespace` label selection, default-to-own-namespace, out-of-allowed-scope ignored) in `internal/controller/selection_test.go`
- [ ] T035 [P] [US3] Unit tests for privilege-boundary validation (controller-ns allowed; other-ns + `selector.namespace` → `ConfigurationError`) in `internal/controller/selection_test.go`
- [ ] T036 [P] [US3] Integration (envtest) test for multi-namespace selection and privilege-boundary rejection in `test/integration/namespace_test.go` (Scenarios F, G)

### Implementation for User Story 3

- [ ] T037 [US3] Implement namespace-scope resolution via `spec.selector.namespace` `metav1.LabelSelector` over `Namespace` objects, defaulting to own namespace, constrained to allowed scope (FR-009/FR-010) in `internal/controller/selection.go`
- [ ] T038 [US3] Implement privilege-boundary enforcement: reject `selector.namespace` for ProgressiveSyncs outside the controller namespace with `phase: Error` + `ConfigurationError` (FR-021, SC-010) in `internal/controller/selection.go`
- [ ] T039 [P] [US3] Author CRD + controller manifests in `config/crd/`, `config/manager/`, `config/configmap/` (Deployment, ServiceAccount, ConfigMap with `--installation-id`/`--application-namespaces`)
- [ ] T040 [P] [US3] Author least-privilege RBAC in `config/rbac/` (Role/ClusterRole scoped to allowed namespaces + Applications read + operation trigger) (FR-011)
- [ ] T041 [US3] Produce aggregated installable `manifests/install.yaml` supporting both topologies (Argo CD namespace and dedicated namespace) (FR-008/FR-009)

**Checkpoint**: Controller is installable in both topologies with safe cross-namespace behavior

---

## Phase 7: User Story 4 - Observe rollout progress and status (Priority: P2)

**Goal**: Surface accurate, real-time status: active step, per-Application sync/health progress, overall state (progressing/waiting/complete), conditions, and bounded rollout history.

**Independent Test**: Trigger a multi-step rollout and read status; assert active step, per-Application progress, waiting reason, completion, and history entries with `Completed`/`Superseded` results (quickstart Scenarios A, E).

### Tests for User Story 4 ⚠️

- [ ] T042 [P] [US4] Unit tests for status/condition computation and history bounding (default last 10) in `internal/controller/status_test.go`
- [ ] T043 [P] [US4] Integration (envtest) test for status accuracy across a rollout and history recording in `test/integration/status_test.go` (Scenarios A, E)

### Implementation for User Story 4

- [ ] T044 [US4] Finalize `conditions[]` management (`Ready`/`Progressing`/`Error`) with `lastTransitionTime`/`observedGeneration` in `internal/controller/status.go` (FR-006)
- [ ] T045 [US4] Implement bounded `history[]` recording with `result` `Completed`/`Superseded` in `internal/controller/status.go` (data-model.md)
- [ ] T046 [US4] Add kubebuilder `printcolumns` (phase, current step, message) to `api/v1alpha1/progressivesync_types.go` and regenerate CRD for `kubectl get` visibility

**Checkpoint**: Operators can fully observe rollout state from resource status

---

## Phase 8: Supersede behavior & conflict detection (Cross-cutting, P1-critical)

**Purpose**: In-flight change handling and overlap safety that span US1/US2/US5

- [ ] T047 [P] Unit tests for supersede logic (new change → grace period for Argo self-refresh → explicit refresh of still-stale Apps → begin only once all refreshed → restart from step 0, prior rollout recorded `Superseded`) in `internal/controller/supersede_test.go` (FR-016)
- [ ] T048 [P] Unit tests for overlapping-selector conflict detection in `internal/controller/selection_test.go` (FR-013)
- [ ] T049 Implement change-candidate detection, configurable grace period (Argo self-refresh window), explicit refresh of still-stale Applications after the grace period, and gating the restart until all affected Applications are refreshed, then supersede + restart from step 0 in `internal/controller/supersede.go` (FR-016; mirrors upstream `ensureApplicationsReconciled`/`checkAllApplicationsReconciled`/`addRefreshAnnotationToApplications`)
- [ ] T050 Implement conflict detection when multiple `ProgressiveSync`s govern the same Application and surface it in status (FR-013)

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Improvements affecting multiple user stories

- [ ] T051 [P] Emit operational Events and Prometheus metrics for rollout lifecycle across the reconciler (FR-015, Principle V)
- [ ] T052 [P] Verify idempotency/fail-safe under retries and add regression tests in `test/integration/idempotency_test.go` (FR-012/FR-014)
- [ ] T053 [P] Add e2e test against a kind cluster with Argo CD in `test/e2e/` exercising quickstart Scenarios A–G
- [ ] T054 [P] Developer docs: README + `docs/` on building, testing, and running the controller
- [ ] T055 Run quickstart.md validation (apply CRD + scenarios) and confirm the design-phase Done checklist

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Setup — BLOCKS all user stories
- **User Stories (Phases 3–7)**: All depend on Foundational completion
  - US1 (P1) → US2 (P1) → US5 (P1) recommended order (US1 is MVP); US2/US5 refine the selection US1 uses
  - US3 (P2) and US4 (P2) can follow; US4 depends on status foundations touched by US1
- **Supersede/conflict (Phase 8)**: Depends on US1 + US2 + US5 (selection & progression must exist)
- **Polish (Phase 9)**: Depends on all targeted stories

### User Story Dependencies

- **US1 (P1)**: Foundational only — the core MVP
- **US2 (P1)**: Foundational; integrates with US1's governed-set consumption
- **US5 (P1)**: Foundational + US2 (filters the selected set)
- **US3 (P2)**: Foundational + US2 (extends selection to namespaces)
- **US4 (P2)**: Foundational + US1 (status has meaning once rollout logic exists)

### Within Each User Story

- Tests written and failing before implementation (Principle III)
- API types (Phase 2) before controller logic
- Selection/steps/status modules before reconciler wiring
- Story complete and independently testable before next priority

### Parallel Opportunities

- Setup tasks T003/T004/T005 in parallel
- Foundational T006, T012, T013 in parallel (distinct files); T007→T008→T009→T010 are sequential (same/generated files)
- All `[P]` test tasks within a story run together before that story's implementation
- Manifests (T039/T040) parallel to controller code in US3
- Different stories can be staffed in parallel once Foundational completes

---

## Parallel Example: User Story 1

```bash
# Write all US1 tests first (parallel), ensure they FAIL:
Task: "Unit tests for step grouping in internal/controller/steps_test.go"          # T014
Task: "Unit tests for success-condition evaluation in internal/controller/steps_test.go"  # T015
Task: "Integration test for ordered progression in test/integration/rollout_test.go"      # T016
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 (Setup) + Phase 2 (Foundational)
2. Complete Phase 3 (US1): ordered, health-gated rollout
3. **STOP and VALIDATE**: run Scenario A independently
4. Demo the MVP

### Incremental Delivery

1. Setup + Foundational → foundation ready
2. US1 → test → demo (MVP: staged rollout)
3. US2 → test → demo (selector-driven membership)
4. US5 → test → demo (tenancy safety)
5. US3 → test → demo (multi-namespace + install manifests)
6. US4 → test → demo (full observability)
7. Phase 8 (supersede/conflict) + Phase 9 (polish) → hardened release

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to a specific user story for traceability
- Verify tests fail before implementing (Principle III, NON-NEGOTIABLE)
- Commit after each task or logical group
- Stop at any checkpoint to validate a story independently
- Avoid: vague tasks, same-file conflicts, cross-story dependencies that break independence
