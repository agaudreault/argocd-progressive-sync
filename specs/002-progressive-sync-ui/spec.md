# Feature Specification: Progressive Sync UI

**Feature Branch**: `002-progressive-sync-ui`

**Created**: 2026-09-15

**Status**: Draft

**Input**: User description: "Similar to how the gitops-promoter has a UI experience for the promotionStrategy, the progressive sync should also have a UI (https://github.com/argoproj-labs/gitops-promoter). The UI should contain the list of the sync (history) with the completion state, any ongoing sync state, and a list of the selected applications so the user can click on them and navigate to the Argo CD app. The UI should also contain a summary of the current ongoing operation and surface any conditions on the progressive sync."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View current progressive sync status at a glance (Priority: P1)

An operator opens the UI for a progressive sync and immediately sees a summary of what is happening
now: whether a rollout is in progress, paused, or complete; which stage is active; and high-level
progress. This gives them confidence about the current state without inspecting raw resources.

**Why this priority**: The primary reason to build a UI is to make the live state of a rollout
legible. Without the current-operation summary, the UI delivers no value over the CLI. It is the
MVP.

**Independent Test**: With a progressive sync mid-rollout, open its UI view and confirm it shows the
overall state, the active stage, and progress that matches the underlying resource status.

**Acceptance Scenarios**:

1. **Given** a progressive sync with an ongoing rollout, **When** the operator opens its UI, **Then**
   a summary of the current operation (state, active stage, progress) is displayed.
2. **Given** a progressive sync with no active rollout, **When** the operator opens its UI, **Then**
   the summary indicates an idle/complete state.
3. **Given** the underlying status changes, **When** the operator is viewing the UI, **Then** the
   summary reflects the new state without requiring a manual page reload beyond normal refresh.

---

### User Story 2 - Browse selected Applications and navigate to Argo CD (Priority: P1)

An operator sees the list of Applications currently selected/governed by the progressive sync, along
with each Application's sync and health state, and can click an Application to navigate directly to
that Application in the Argo CD UI.

**Why this priority**: Understanding which Applications are in scope, and jumping to them in Argo CD,
is core to operating and troubleshooting a rollout. It is essential alongside the status summary.

**Independent Test**: For a progressive sync selecting several Applications, open the UI and confirm
all governed Applications are listed with their state, and that clicking one navigates to the
corresponding Argo CD Application view.

**Acceptance Scenarios**:

1. **Given** a progressive sync governing several Applications, **When** the operator opens the UI,
   **Then** all selected Applications are listed with sync/health state.
2. **Given** the list of Applications, **When** the operator clicks an Application, **Then** they are
   taken to that Application's page in the Argo CD UI.
3. **Given** membership changes (an Application is added or removed), **When** the operator views the
   list, **Then** it reflects the current governed set.

---

### User Story 3 - Review sync history with completion states (Priority: P2)

An operator reviews a history of past progressive syncs/rollouts for a given progressive sync,
seeing when each ran and how it completed (succeeded, failed, paused/aborted). This helps them audit
and understand rollout behavior over time.

**Why this priority**: History is valuable for auditing and diagnosing recurring issues, but the
live current-state views (P1) deliver the primary value first.

**Independent Test**: After running multiple rollouts (including at least one failure), open the UI
history view and confirm each past rollout appears with its completion state and timing.

**Acceptance Scenarios**:

1. **Given** multiple past rollouts, **When** the operator opens the history view, **Then** each
   rollout is listed with its completion state and timing.
2. **Given** a failed past rollout, **When** the operator inspects its history entry, **Then** the
   failure and, where available, its reason are shown.
3. **Given** an in-progress rollout, **When** the operator views history, **Then** the ongoing
   rollout is distinguishable from completed ones.

---

### User Story 4 - Surface conditions and diagnostics (Priority: P2)

An operator sees the conditions reported on the progressive sync (e.g., progressing, paused, error,
tenancy/validation issues) surfaced in the UI, so they can understand why a rollout is in its
current state and what, if anything, needs attention.

**Why this priority**: Conditions turn a status display into an actionable diagnostic tool, but they
build on the live status summary (P1) already existing.

**Independent Test**: Put a progressive sync into a paused/error state and confirm the UI surfaces
the relevant condition(s) with human-readable messages.

**Acceptance Scenarios**:

1. **Given** a progressive sync reporting conditions, **When** the operator opens the UI, **Then**
   those conditions are displayed with their status and message.
2. **Given** a rollout halted due to a failed stage, **When** the operator views conditions, **Then**
   the reason for the halt is clearly surfaced.
3. **Given** a healthy, progressing rollout, **When** the operator views conditions, **Then** the UI
   indicates a normal/progressing state without spurious warnings.

---

### Edge Cases

- What happens when a progressive sync has no history yet? The history view shows an empty state
  rather than an error.
- What happens when a selected Application no longer exists (deleted)? The UI indicates the
  Application is gone rather than presenting a broken navigation link.
- How does the UI behave when the Argo CD UI location for an Application is unknown or the
  Application lives in a different namespace/instance? The navigation target must be resolved
  correctly or the link disabled with an explanation.
- How does the UI handle a very large governed set (hundreds of Applications)? It must remain usable
  (e.g., pagination/virtualized list) without overwhelming the operator.
- What happens when the backing status data is temporarily unavailable? The UI shows a clear loading
  or error state instead of stale data presented as current.
- How does the UI handle a user without permission to view a selected Application in Argo CD?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: UI MUST display a summary of the current operation for a progressive sync, including
  overall state (progressing, paused, complete/idle), active stage, and progress.
- **FR-002**: UI MUST list the Applications currently selected/governed by a progressive sync, each
  with its sync and health state.
- **FR-003**: UI MUST allow the operator to navigate from a listed Application to that Application's
  view in the Argo CD UI.
- **FR-004**: UI MUST display a history of past rollouts for a progressive sync, each with a
  completion state (succeeded, failed, paused/aborted) and timing.
- **FR-005**: UI MUST distinguish an ongoing rollout from completed ones in both the summary and
  history views.
- **FR-006**: UI MUST surface the conditions reported on a progressive sync, including status and
  human-readable message.
- **FR-007**: UI MUST reflect updates to progressive sync state within a reasonable refresh interval
  without requiring the operator to manually reload.
- **FR-008**: UI MUST present clear empty, loading, and error states rather than showing stale or
  broken content.
- **FR-009**: UI MUST handle deleted or inaccessible Applications gracefully (indicate the condition;
  do not present broken links).
- **FR-010**: UI MUST remain usable for large governed sets (e.g., via pagination or an equivalent
  approach).
- **FR-011**: UI MUST only expose information the viewing user is authorized to see, consistent with
  the surrounding Argo CD access model.

### Key Entities *(include if feature involves data)*

- **Progressive Sync (view)**: The resource whose live state, selected Applications, conditions, and
  history are presented. Owned by feature 001; this feature reads and displays it.
- **Rollout History Entry**: A record of a past (or in-progress) rollout for a progressive sync,
  including completion state and timing.
- **Selected Application (view)**: An Argo CD Application governed by the progressive sync, shown with
  its sync/health state and a navigation target into the Argo CD UI.
- **Condition (view)**: A reported condition on the progressive sync (type, status, reason, message)
  surfaced for diagnostics.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An operator can determine the current state of a progressive sync (progressing/paused/
  complete, active stage, progress) from the UI within seconds of opening it, without inspecting raw
  resources.
- **SC-002**: An operator can see all governed Applications and reach the corresponding Argo CD
  Application view in a single click, for 100% of existing, accessible Applications.
- **SC-003**: An operator can review past rollouts and identify each one's completion state and when
  it occurred.
- **SC-004**: When a rollout is halted or errored, the operator can identify the reason from the
  surfaced conditions without external tools.
- **SC-005**: The UI reflects a change in progressive sync state within one refresh interval of the
  change occurring.
- **SC-006**: The UI remains responsive and navigable with at least several hundred governed
  Applications.

## Assumptions

- This UI presents data produced by the progressive sync controller (feature 001); it does not
  itself drive rollouts. Read-only presentation is the scope for v1 (no start/pause/abort controls
  unless added later).
- The experience is modeled conceptually on the GitOps Promoter UI for PromotionStrategy
  ([argoproj-labs/gitops-promoter](https://github.com/argoproj-labs/gitops-promoter)), adapted to
  progressive sync concepts (stages, selected Applications, conditions, history).
- Navigation targets an existing Argo CD UI; the UI can resolve an Application's Argo CD location
  (namespace/instance aware) to build the link.
- Rollout history is available from the controller's reported status/records; the retention depth and
  storage mechanism are finalized during planning.
- Authorization aligns with the surrounding Argo CD access model rather than introducing a separate
  permission system.
- Whether this ships as an extension/embedded view within the Argo CD UI or as a standalone UI is an
  implementation decision deferred to planning; the requirements here are presentation-focused and
  agnostic to that choice.
