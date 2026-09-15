# Feature Specification: Documentation

**Feature Branch**: `003-documentation`

**Created**: 2026-09-15

**Status**: Draft

**Input**: User description: "The documentation should be in markdown, concise and clear. We should have a product documentation that explain to the users how to configure things, and what is the expected impact/effect of these configurations. The user facing documentation should not contain technical information on how the system works under the hood. We should also have an operator documentation space that explain how to deploy and configure the controller."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Configure progressive sync as a product user (Priority: P1)

An application/platform user who consumes the progressive sync capability reads the product
documentation to learn how to configure a progressive sync: what options exist, how to set them, and
— most importantly — what effect each option has on their rollouts. The documentation stays at the
level of intent and outcome and does not explain internal mechanics.

**Why this priority**: The product documentation is what lets users successfully adopt and configure
the feature. Without it, the capability is unusable to its intended audience. It is the MVP of this
documentation effort.

**Independent Test**: Give the product docs to a user unfamiliar with the internals and confirm they
can configure a progressive sync correctly and correctly predict the effect of their configuration,
without needing to read source code or operator docs.

**Acceptance Scenarios**:

1. **Given** the product documentation, **When** a user looks up a configuration option, **Then**
   they find how to set it and a clear description of its impact/effect.
2. **Given** the product documentation, **When** a user reads it end to end, **Then** it contains no
   under-the-hood technical/implementation detail.
3. **Given** a common configuration goal, **When** a user follows the docs, **Then** they can achieve
   that goal without external help.

---

### User Story 2 - Deploy and configure the controller as an operator (Priority: P1)

An operator responsible for running the controller reads the operator documentation to learn how to
deploy it and configure its runtime behavior (namespaces, installation topology, permissions, and
operational settings). The docs give them everything needed to get a working, correctly-scoped
installation.

**Why this priority**: Operators cannot run the controller without deployment/configuration
guidance. This is equally essential to product docs and forms the second half of the MVP.

**Independent Test**: Give the operator docs to someone who has not deployed the controller before
and confirm they can install and configure it (including a multi-namespace topology) using only the
docs.

**Acceptance Scenarios**:

1. **Given** the operator documentation, **When** an operator follows the deployment guidance,
   **Then** they can install the controller in a supported topology.
2. **Given** the operator documentation, **When** an operator needs to change runtime configuration
   (e.g., watched namespaces), **Then** the docs explain each setting and its effect.
3. **Given** the operator documentation, **When** an operator installs, **Then** the guidance covers
   both the Argo CD-namespace and dedicated-namespace/multi-namespace deployments.

---

### User Story 3 - Find the right documentation quickly (Priority: P2)

A reader can quickly tell which documentation space (product vs. operator) is relevant to them and
navigate to the specific topic they need, because the docs are clearly organized, concise, and
consistently structured.

**Why this priority**: Discoverability multiplies the value of the content, but the content itself
(US1/US2) must exist first.

**Independent Test**: Ask readers with different roles to locate a specific topic and confirm they
reach the correct page quickly without wandering between the two spaces.

**Acceptance Scenarios**:

1. **Given** the documentation set, **When** a reader arrives, **Then** the split between product
   (user) and operator documentation is clear and labeled.
2. **Given** a specific question, **When** a reader searches or browses, **Then** they can locate the
   relevant topic within a few navigation steps.
3. **Given** any page, **When** a reader views it, **Then** it follows a consistent, concise
   structure.

---

### Edge Cases

- What happens when a configuration option is relevant to both users and operators? It must be
  presented in the appropriate space (or cross-linked) without leaking implementation detail into the
  product docs.
- How is documentation kept consistent with the actual configuration options as they evolve? Stale or
  missing options should be detectable.
- How does a user distinguish product docs from operator docs if they land deep-linked on a single
  page? Each page should make its audience/space evident.
- How are deprecated or version-specific options communicated so users don't apply outdated guidance?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Documentation MUST be authored in Markdown.
- **FR-002**: Documentation MUST be concise and clear.
- **FR-003**: Documentation MUST include a **product (user-facing)** space that explains how to
  configure the progressive sync capability and the expected impact/effect of each configuration.
- **FR-004**: The product (user-facing) documentation MUST NOT contain technical information about how
  the system works under the hood.
- **FR-005**: Documentation MUST include an **operator** space that explains how to deploy and
  configure the controller.
- **FR-006**: The operator documentation MUST cover both supported deployment topologies (within the
  Argo CD namespace, and a dedicated namespace with multi-namespace discovery).
- **FR-007**: The two documentation spaces MUST be clearly distinguished so readers know which
  audience each page serves.
- **FR-008**: Each configuration option documented for users MUST describe both how to set it and its
  effect/impact.
- **FR-009**: Documentation MUST be organized for quick navigation to a specific topic.
- **FR-010**: Documentation MUST follow a consistent structure and style across pages.
- **FR-011**: Documentation MUST indicate version-specific or deprecated guidance where applicable so
  readers do not apply outdated instructions.

### Key Entities *(include if feature involves data)*

- **Product Documentation (space)**: The user-facing documentation set. Explains configuration and
  its effects for users; excludes internal mechanics.
- **Operator Documentation (space)**: The documentation set for those deploying/running the
  controller. Explains installation, topologies, and runtime configuration.
- **Documentation Page**: An individual Markdown topic belonging to one space, following a consistent
  structure and clearly labeled with its audience.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user unfamiliar with the internals can correctly configure a progressive sync and
  predict its effect using only the product documentation.
- **SC-002**: An operator new to the controller can deploy and configure it in a supported topology
  using only the operator documentation.
- **SC-003**: 100% of user-documented configuration options include both how to set them and their
  effect/impact.
- **SC-004**: The product documentation contains zero under-the-hood implementation details (verified
  by review).
- **SC-005**: Readers can identify which space (product vs. operator) a page belongs to on 100% of
  pages.
- **SC-006**: A reader can locate a specific topic within a few navigation steps.

## Assumptions

- Documentation is written in Markdown and lives in the repository (a `docs/` area is the assumed
  location; the exact site-generation tooling is deferred to planning and does not affect these
  requirements).
- "Product" documentation targets users who configure and consume progressive sync; "operator"
  documentation targets those who deploy and run the controller. These are the two distinct audiences.
- The documented configuration surface corresponds to the capabilities defined in feature 001
  (Progressive Sync Controller) and, where relevant, feature 002 (Progressive Sync UI).
- Concise/clear is interpreted as task-focused pages with consistent structure, minimal redundancy,
  and plain language — measured via review rather than a strict word count.
- Reference/example manifests may appear in operator docs; deep architectural/internal design
  explanation is out of scope for both spaces (product docs strictly exclude it; operator docs focus
  on deploy/configure rather than internals).
