# Feature Specification: Shared-Contracts Schema Source of Truth

**Feature Branch**: `001-shared-contracts-schemas`

**Created**: 2026-06-07

**Status**: Draft

**Input**: User description: "As a developer working on product-service or order-service, I need shared-contracts to provide gRPC proto definitions, Kafka event schemas, and HTTP DTOs so that all services have a single versioned source of truth for inter-service communication contracts"

## Clarifications

### Session 2026-06-07

- Q: What is the deliverable boundary of this feature? → A: Define & publish only —
  `shared-contracts` holds proto/schemas/DTOs plus generated typed artifacts and
  versioning/changelog. Actual gRPC servers and Kafka producer/consumer wiring in the
  services are out of scope (separate features).
- Q: Which consumers must `shared-contracts` generate typed artifacts for? → A: Java/JVM
  only (the Spring Boot services). TypeScript/Angular-frontend type generation is out of
  scope.
- Q: How is backward-compatibility enforced when a contract changes? → A: Automated CI gate
  — a compatibility check runs in CI, fails the build on an undeclared breaking change, and
  enforces the matching version bump.
- Q: Across how many versions must a consumer remain interoperable? → A: N and N-1 — a
  service on the latest version must interoperate with peers one compatible version behind
  during rollout.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consume contracts as a single versioned dependency (Priority: P1)

A developer building `product-service` or `order-service` needs the data types,
message shapes, and error models used to communicate with other parts of the platform.
Instead of hand-copying type definitions between services, they declare a dependency on
`shared-contracts` at a specific version and obtain every contract artifact — gRPC
service/message definitions, Kafka event schemas, and HTTP request/response DTOs — from
that one source.

**Why this priority**: Without a consumable single source of truth, services drift apart
and integration breaks. This is the foundational capability the entire feature exists to
deliver; everything else builds on it.

**Independent Test**: Add the `shared-contracts` dependency at a pinned version to a
service, reference a DTO, a proto-defined message, and an event schema from that service's
code, and confirm the service builds successfully using only the published artifact (no
copied or duplicated definitions).

**Acceptance Scenarios**:

1. **Given** `shared-contracts` is published at a known version, **When** a developer adds
   it as a dependency at that version, **Then** they can reference HTTP DTOs, gRPC message
   types, and Kafka event schemas without redefining them locally.
2. **Given** a service already depends on `shared-contracts`, **When** the developer
   inspects the artifact, **Then** all three contract categories (gRPC, Kafka, HTTP) are
   present and discoverable in one place.
3. **Given** two services depend on the same `shared-contracts` version, **When** they
   exchange a message defined in that version, **Then** both interpret the message
   identically with no field mismatch.

---

### User Story 2 - Generate usable code from contract definitions (Priority: P2)

A developer needs language-appropriate, ready-to-use types generated from the canonical
gRPC proto and event schema definitions so they can call and serve contracts without
manually writing serialization code.

**Why this priority**: A source of truth that cannot be turned into working code forces
duplication and defeats the purpose. High value, but depends on US1 being in place first.

**Independent Test**: From a clean checkout, run the documented build of `shared-contracts`
and confirm consumable typed artifacts are produced for the gRPC definitions and event
schemas, and that a service can compile against them.

**Acceptance Scenarios**:

1. **Given** a proto definition exists in `shared-contracts`, **When** the artifact is
   built, **Then** corresponding typed message/stub artifacts are produced and available to
   consuming services.
2. **Given** a Kafka event schema exists in `shared-contracts`, **When** the artifact is
   built, **Then** a typed representation of that event is produced for producers and
   consumers.
3. **Given** an HTTP DTO is defined, **When** a service serializes and deserializes it,
   **Then** the round-trip preserves all fields and the RFC 7807 error model is available
   for error responses.

---

### User Story 3 - Detect breaking changes through versioning (Priority: P3)

A developer changing a shared contract needs to know whether the change is backward
compatible, and consuming teams need a clear signal (version number + changelog) when a
breaking change is published.

**Why this priority**: Versioning discipline prevents silent breakage across independently
deployed services, but the platform can begin consuming contracts before automated
compatibility checks exist.

**Independent Test**: Make a backward-incompatible change to a contract, run the documented
compatibility check, and confirm it flags the change and that the published version reflects
a breaking bump with a corresponding changelog entry.

**Acceptance Scenarios**:

1. **Given** a contract is modified in a backward-incompatible way, **When** the change is
   validated, **Then** the process flags it as breaking and requires a major version bump.
2. **Given** a new `shared-contracts` version is published, **When** a consumer reviews the
   release, **Then** a changelog identifies what changed and whether it is breaking.
3. **Given** a backward-compatible field addition, **When** the change is validated, **Then**
   it is permitted with a minor/patch version bump and existing consumers continue to build.

---

### Edge Cases

- What happens when two services temporarily depend on different `shared-contracts`
  versions that define the same message differently? Interoperability is guaranteed only
  across N and N-1 (see FR-010a); spans wider than one version are not supported.
- How does the system handle a contract definition that is syntactically invalid or
  references a type that does not exist?
- What happens when an HTTP DTO, a gRPC message, and a Kafka event are all expected to
  represent the same domain concept — how is consistency between them ensured?
- How does a consumer behave when it receives a message containing fields it does not yet
  recognize (forward compatibility)?
- What is the expected behavior when a breaking change is published without the required
  version bump?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `shared-contracts` MUST provide gRPC proto definitions for inter-service
  communication contracts.
- **FR-002**: `shared-contracts` MUST provide Kafka event schemas for asynchronous
  inter-service events.
- **FR-003**: `shared-contracts` MUST provide HTTP request/response DTOs.
- **FR-004**: `shared-contracts` MUST provide the RFC 7807 Problem Details error response
  model used across the platform.
- **FR-005**: All three contract categories (gRPC, Kafka, HTTP) MUST reside in the single
  `shared-contracts` artifact, serving as the one source of truth.
- **FR-006**: `shared-contracts` MUST be consumable by `product-service` and
  `order-service` as a pinned, versioned dependency rather than via copied definitions.
- **FR-007**: Each published release of `shared-contracts` MUST carry a single explicit
  version identifier covering all contract categories together.
- **FR-008**: The build of `shared-contracts` MUST produce typed, usable Java/JVM artifacts
  derived from the gRPC and event-schema definitions for consumption by the Spring Boot
  services. Generating artifacts for non-JVM consumers (e.g., the Angular frontend) is out
  of scope.
- **FR-009**: Any backward-incompatible change to a contract MUST require a major version
  bump.
- **FR-009a**: An automated compatibility check MUST run in CI on every change to a contract
  definition, MUST fail the build when an undeclared backward-incompatible change is detected,
  and MUST enforce that the version bump matches the detected change severity.
- **FR-010**: Backward-compatible changes MUST be publishable under a minor or patch version
  bump without breaking existing consumers.
- **FR-010a**: A consumer on the latest version MUST remain interoperable with peers running
  the immediately preceding compatible version (N and N-1) during a rollout window, so that
  services can deploy independently.
- **FR-011**: Each release MUST include a changelog identifying what changed and whether the
  change is breaking.
- **FR-012**: The contract definitions MUST be discoverable in one well-known location within
  the artifact so developers can find every contract without searching multiple services.
- **FR-013**: A defined HTTP endpoint contract in `shared-contracts` MUST remain consistent
  with its documentation in `api-contract.md` (per the project constitution).
- **FR-014**: Invalid or unresolvable contract definitions MUST cause the `shared-contracts`
  build to fail rather than publish a broken artifact.

### Key Entities *(include if feature involves data)*

- **Contract Artifact**: The single published, versioned unit of `shared-contracts`
  containing all contract categories; attributes include version identifier and changelog.
- **gRPC Proto Definition**: A definition of a service method and its message types used for
  synchronous inter-service calls.
- **Kafka Event Schema**: The shape of an asynchronous event published/consumed between
  services; includes event name, version, and payload fields.
- **HTTP DTO**: A request or response data shape exchanged over HTTP, including the shared
  RFC 7807 error model.
- **Version Identifier**: The semantic version applied to a release of the contract artifact,
  signaling compatibility expectations to consumers.
- **Changelog Entry**: A human-readable record of what changed in a release and whether it is
  breaking.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer can add `shared-contracts` to a service and reference a contract
  from each of the three categories in under 15 minutes, with no manually duplicated
  definitions.
- **SC-002**: 100% of inter-service contract types used by `product-service` and
  `order-service` originate from `shared-contracts` (zero locally redefined contract types).
- **SC-003**: Every published release carries exactly one version identifier and an
  accompanying changelog entry.
- **SC-004**: 100% of backward-incompatible contract changes are released under a major
  version bump (no breaking change ships under a minor/patch version).
- **SC-005**: Two services pinned to the same contract version exchange messages with a 0%
  field-mismatch rate in integration testing.
- **SC-006**: A contract definition error is detected at build time in 100% of cases and
  never reaches a published artifact.
- **SC-007**: An undeclared backward-incompatible contract change is blocked by the CI
  compatibility gate in 100% of cases before merge.

## Assumptions

- The "users" of this feature are platform developers (primarily working on
  `product-service` and `order-service`), not end customers.
- `shared-contracts` continues to be distributed as a library installed to the local
  artifact repository during development, consistent with the project constitution.
- Introducing gRPC proto definitions and Kafka event schemas represents an intended
  expansion of inter-service communication mechanisms beyond the constitution's current
  "HTTP REST through api-gateway only" wording; the constitution and `api-contract.md` will
  be updated to reflect gRPC and Kafka as sanctioned inter-service mechanisms. This spec
  documents the contract-artifact requirements regardless of the transport decision.
- Semantic versioning is the compatibility scheme (major = breaking, minor/patch =
  compatible), consistent with the constitution's versioning policy.
- A single shared version covers all contract categories together; categories are not
  versioned independently.
- Backward compatibility for consumers is expected during a transition window when adjacent
  versions are deployed (specifically N and N-1), allowing independent service deployment.

### Out of Scope

- Running gRPC servers/clients or Kafka producers/consumers inside any service. This feature
  defines and publishes the contracts and their generated typed artifacts only; transport
  adoption in `product-service`/`order-service` is a separate feature.
- Migrating existing HTTP/REST inter-service calls to gRPC or Kafka.
- Generating typed artifacts for non-JVM consumers (e.g., TypeScript types for the Angular
  frontend).
