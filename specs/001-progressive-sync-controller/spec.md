# Feature Specification: Progressive Sync Controller

**Feature Branch**: `001-progressive-sync-controller`

**Created**: 2026-09-15

**Status**: Draft

**Input**: User description: "I want to create a new kubernetes controller. It's role will be to manage the progressive sync of Argo CD application. This pattern is already implemented in the ApplicationSet controller, but the logic should be extracted to it's own controller. As part of this work, we need to define the CRD spec, and how it integrates with the Application. Usually, it is a common pattern to use a selector. The project should follow the standard Argo CD structure and golang preferences. It should contain manifest to deploy the controller, with all the rbac, configMap and other associated resources. It can be deployed in Argo CD namespace, or it can be deployed in it's own namespace and discover argo applications in multiple namespace."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Roll out changes to a group of Applications in ordered stages (Priority: P1)

A platform operator manages many Argo CD Applications that represent the same workload across
different targets (clusters, regions, or environments). Instead of letting all Applications sync at
once when a change is detected, the operator wants the change to roll out in ordered stages — for
example, sync one canary Application first, wait until it is healthy, then proceed to the next
group, and finally to the rest. If any stage does not become healthy, the rollout must pause so the
operator can intervene before the change reaches the remaining targets.

**Why this priority**: This is the core value of the feature. Without staged, health-gated
rollouts across a selected set of Applications, the controller provides no benefit over normal Argo
CD auto-sync. It is the minimum viable product.

**Independent Test**: Create a progressive sync resource that selects a set of test Applications and
defines two ordered stages. Introduce a change and confirm the first stage syncs and must reach a
healthy state before the second stage begins, and that a failed first stage halts progression.

**Acceptance Scenarios**:

1. **Given** a progressive sync resource selecting several Applications with two ordered stages,
   **When** a change makes the Applications out of sync, **Then** only the first stage's
   Applications are synced initially.
2. **Given** the first stage has synced and become healthy, **When** the controller re-evaluates,
   **Then** the next stage's Applications are synced.
3. **Given** the first stage syncs but fails to become healthy, **When** the controller
   re-evaluates, **Then** progression halts and no further stages are synced.
4. **Given** all stages have synced and are healthy, **When** the controller re-evaluates, **Then**
   the rollout is reported as complete.

---

### User Story 2 - Select which Applications participate via a selector (Priority: P1)

An operator defines which Applications a progressive sync governs using a selector (for example,
matching labels on Applications) rather than listing each Application by name. As Applications that
match the selector are added or removed, the set governed by the progressive sync updates
automatically.

**Why this priority**: Selector-based targeting is how operators express intent at scale and is a
prerequisite for staged rollouts to be useful beyond a hand-maintained list. It is essential to the
MVP alongside User Story 1.

**Independent Test**: Create a progressive sync resource with a label selector, then create and
label Applications so some match and some do not. Confirm only matching Applications are included,
and that adding/removing a matching label changes membership.

**Acceptance Scenarios**:

1. **Given** a progressive sync resource with a label selector, **When** Applications exist that
   match and do not match, **Then** only matching Applications are governed.
2. **Given** a governed set, **When** a new Application is created that matches the selector,
   **Then** it becomes part of the governed set on the next evaluation.
3. **Given** a governed Application, **When** its labels change so it no longer matches, **Then** it
   is removed from the governed set.

---

### User Story 3 - Deploy the controller flexibly across namespaces (Priority: P2)

An operator installs the controller either inside the Argo CD namespace or in its own dedicated
namespace. When running in its own namespace, the controller can discover and govern Applications
that live in multiple namespaces, according to configuration. The controller ships with everything
needed to deploy it.

**Why this priority**: Flexible deployment topology and multi-namespace discovery are important for
real-world adoption but are not required to demonstrate the core staged-rollout value, so they rank
below the core P1 behavior.

**Independent Test**: Apply the provided deployment manifests in (a) the Argo CD namespace and (b) a
separate namespace configured to watch multiple Application namespaces, and confirm the controller
governs the intended Applications in each topology.

**Acceptance Scenarios**:

1. **Given** the provided manifests, **When** applied in the Argo CD namespace, **Then** the
   controller runs and governs Applications in that namespace.
2. **Given** the provided manifests, **When** applied in a dedicated namespace configured to watch
   multiple namespaces, **Then** the controller governs matching Applications across those
   namespaces.
3. **Given** an installation, **When** the controller starts, **Then** it has exactly the
   permissions it needs and no more.

---

### User Story 4 - Observe rollout progress and status (Priority: P2)

An operator inspects a progressive sync resource to understand current rollout state: which stage is
active, which Applications have synced, which are healthy, and whether the rollout is progressing,
paused, or complete.

**Why this priority**: Visibility is critical for trust and troubleshooting, but the rollout logic
itself (P1) must exist first for status to have meaning.

**Independent Test**: Trigger a multi-stage rollout and read the progressive sync resource's status,
confirming it accurately reflects the active stage, per-Application progress, and overall state as
the rollout advances.

**Acceptance Scenarios**:

1. **Given** an in-progress rollout, **When** the operator inspects the resource, **Then** status
   shows the active stage and per-Application sync/health progress.
2. **Given** a halted rollout, **When** the operator inspects the resource, **Then** status
   indicates the rollout is paused and why.
3. **Given** a completed rollout, **When** the operator inspects the resource, **Then** status
   reports completion.

---

### User Story 5 - Restrict selection to Applications of a single Argo CD instance (Priority: P1)

In environments where more than one Argo CD instance runs on the same cluster(s), or where a
progressive sync controller watches multiple namespaces, an operator must be certain that a
progressive sync only governs Applications belonging to one Argo CD installation. The controller
validates the Argo CD installation identity recorded on each Application and refuses to include
Applications that belong to a different instance, preventing one Argo CD's rollout from acting on
another Argo CD's Applications.

**Why this priority**: This is a tenancy/safety guarantee. Without it, a selector in a
multi-instance or multi-namespace environment could inadvertently capture and sync Applications
owned by a different Argo CD installation — a correctness and isolation violation. It is essential
to the MVP because selection (User Story 2) and multi-namespace discovery (User Story 3) are unsafe
without it.

**Independent Test**: With Applications from two different Argo CD installations present (bearing
different installation identities), create a progressive sync bound to one installation and confirm
only that installation's Applications are governed and the others are excluded and reported.

**Acceptance Scenarios**:

1. **Given** Applications from two Argo CD installations that both match a selector, **When** the
   progressive sync evaluates membership, **Then** only Applications matching the controller's own
   Argo CD installation identity are governed.
2. **Given** a selected Application whose installation identity does not match, **When** membership
   is evaluated, **Then** the Application is excluded and the mismatch is surfaced in status.
3. **Given** a selected Application that is missing an installation identity, **When** membership is
   evaluated, **Then** the Application is excluded (fail-safe) and the reason is surfaced.

---

### Edge Cases

- What happens when the selector matches zero Applications? The rollout should report an empty,
  successfully-complete (or idle) state rather than erroring.
- What happens when an Application is deleted mid-rollout? It should be removed from the governed set
  and must not block progression.
- What happens when a stage never becomes healthy? Progression must halt indefinitely (subject to
  configurable timeout behavior) rather than proceeding.
- How does the system handle a new change arriving while a rollout is already in progress? The
  behavior for restarting or superseding the in-flight rollout must be defined.
- How does the system handle two progressive sync resources whose selectors overlap on the same
  Application? Conflicting ownership must be detected and surfaced rather than silently fighting.
- What happens when the controller lacks permission to read or act on an Application in a watched
  namespace? It must surface a clear error and not partially apply changes.
- How does the system behave if a governed Application is manually synced out-of-band during a
  rollout?
- What happens when a selected Application carries no Argo CD installation identity, or an identity
  that does not match the controller's? It must be excluded from the governed set and the exclusion
  surfaced, never silently synced.
- How does the system determine its own Argo CD installation identity, and what happens if that
  identity is unknown or misconfigured? It must fail safe and refuse to govern any Application rather
  than guess.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide a custom resource that defines a progressive sync policy,
  including the set of stages and their order.
- **FR-002**: System MUST allow the governed set of Applications to be defined via a selector rather
  than requiring each Application to be named explicitly.
- **FR-003**: System MUST sync the Applications in a stage only after the prior stage has reached a
  configured success condition (e.g., synced and healthy).
- **FR-004**: System MUST halt progression to subsequent stages when a stage fails to meet its
  success condition.
- **FR-005**: System MUST re-evaluate governed membership as Applications matching the selector are
  created, changed, or deleted.
- **FR-006**: System MUST expose observable status including the active stage, per-Application
  sync/health progress, and overall rollout state (progressing, paused, complete).
- **FR-007**: System MUST preserve semantics compatible with the progressive sync behavior currently
  implemented in the ApplicationSet controller, or explicitly document any intentional divergence.
- **FR-008**: System MUST be deployable via provided declarative manifests that include the custom
  resource definition, the controller workload, its identity/permissions, and its configuration.
- **FR-009**: System MUST support deployment both within the Argo CD namespace and within its own
  dedicated namespace.
- **FR-010**: System MUST support discovering and governing Applications across multiple namespaces
  when configured to do so, with the set of watched namespaces being configuration-driven.
- **FR-011**: System MUST operate with least-privilege permissions scoped to only what it needs.
- **FR-012**: System MUST behave idempotently and safely under retries, never leaving a rollout in
  an inconsistent state.
- **FR-013**: System MUST detect and surface conflicts when multiple progressive sync resources
  attempt to govern the same Application.
- **FR-014**: System MUST fail safe — on ambiguity or error it halts progression rather than
  continuing to roll out changes.
- **FR-015**: System MUST emit operational signals (events and metrics) sufficient to observe and
  troubleshoot rollouts.
- **FR-016**: System MUST define behavior for a new change that arrives while a rollout is already
  in progress.
- **FR-017**: System MUST validate the Argo CD installation identity recorded on each candidate
  Application's metadata and include only Applications whose installation identity matches the Argo
  CD installation the controller is bound to.
- **FR-018**: System MUST exclude any selected Application whose installation identity is missing or
  does not match, and MUST surface the exclusion and its reason in status.
- **FR-019**: System MUST determine the Argo CD installation identity it is bound to from
  configuration, and MUST fail safe (govern no Applications) when that identity cannot be
  determined.
- **FR-020**: System MUST ensure all Applications governed by a single progressive sync belong to
  the same Argo CD installation.

### Key Entities *(include if feature involves data)*

- **Progressive Sync**: The policy resource an operator creates. Represents which Applications are
  governed (via selector), the ordered stages of the rollout, the success condition that gates
  progression, and the observed status of the rollout.
- **Stage**: An ordered step within a Progressive Sync. Represents a subset of the governed
  Applications (e.g., by count, percentage, or a finer selector) that are synced together before the
  next stage begins.
- **Application (external)**: The existing Argo CD Application resource that the controller reads and
  triggers to sync. Not owned by this feature; referenced/selected by it. Key attributes used:
  labels (for selection), sync state, health state, and the Argo CD installation identity recorded
  in its metadata (used for tenancy validation).
- **Argo CD Installation Identity**: The identifier of the Argo CD installation that manages an
  Application, recorded in the Application's metadata. Used to ensure a progressive sync only governs
  Applications belonging to a single, expected Argo CD instance.
- **Controller Configuration**: The settings that determine the controller's operating scope, most
  importantly which namespace(s) it watches for Applications and the Argo CD installation identity
  it is bound to.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An operator can define a staged rollout across a selected set of Applications and, when
  a change is introduced, observe that stages execute strictly in order with each stage gated on the
  previous stage's success.
- **SC-002**: A failing stage halts the rollout before any later stage's Applications are changed, in
  100% of failure cases.
- **SC-003**: Changing an Application's labels changes its membership in the governed set on the next
  evaluation, with no manual edits to the progressive sync resource required.
- **SC-004**: The provided manifests install a working controller in both supported topologies
  (Argo CD namespace and dedicated multi-namespace) without manual permission editing.
- **SC-005**: At any point during a rollout, an operator can determine the active stage, which
  Applications have synced, which are healthy, and the overall rollout state from the resource's
  status.
- **SC-006**: Progressive sync behavior produces outcomes equivalent to the existing ApplicationSet
  progressive sync for the same inputs, except where a divergence is explicitly documented.
- **SC-007**: When the selector matches no Applications, the system reports an idle/complete state
  and does not error.
- **SC-008**: In an environment with Applications from more than one Argo CD installation, a
  progressive sync governs only Applications belonging to its bound installation, excluding all
  others in 100% of cases.
- **SC-009**: Every Application governed by a given progressive sync belongs to the same Argo CD
  installation, verifiable from the resource's status.

## Assumptions

- The target environment is a Kubernetes cluster with Argo CD installed and Argo CD Application
  resources present; the controller governs those Applications rather than replacing Argo CD's sync
  engine.
- "Progressive sync" refers to the same conceptual capability offered today by the ApplicationSet
  controller's progressive sync (rollout) strategy, which this feature extracts into a standalone
  controller.
- The success condition that gates stage progression defaults to Applications being both synced and
  healthy, unless configured otherwise.
- Selection defaults to matching Application labels; the exact selectable fields will be finalized
  during planning.
- Multi-namespace discovery is opt-in and governed by configuration, defaulting to a restricted
  scope for safety.
- The controller does not modify Application spec/desired state; it orchestrates when Applications
  are triggered to sync.
- Coexistence with ApplicationSet's own progressive sync on the same Applications is out of scope for
  v1; a given Application is expected to be governed by only one progressive-sync mechanism.
- Argo CD records an installation identity in Application metadata (e.g., a tracking/installation-id
  annotation) that can be read to distinguish which Argo CD instance manages a given Application; the
  exact metadata key is confirmed during planning.
- The controller is bound to exactly one Argo CD installation identity, supplied via configuration
  (or discoverable from the Argo CD installation it is co-deployed with); cross-instance governance
  is explicitly disallowed.
