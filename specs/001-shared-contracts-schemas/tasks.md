---
description: "Task list for shared-contracts schema source of truth"
---

# Tasks: Shared-Contracts Schema Source of Truth

**Input**: Design documents from `/specs/001-shared-contracts-schemas/`

**Prerequisites**: plan.md, spec.md, data-model.md, contracts/, research.md, quickstart.md

**Tests**: INCLUDED — mandatory per constitution Principle III (TDD, 80% line coverage on
hand-written code). For each contract, the test task precedes its implementation task.

**Organization**: Tasks are grouped by user story (US1, US2, US3) so each story is an
independently testable increment.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: US1 / US2 / US3 (omitted for Setup, Foundational, Polish)
- All paths are relative to the repository root.

## Path Conventions

- Module root: `shared-contracts/`
- Proto IDL: `shared-contracts/src/main/proto/`
- Hand-written Java: `shared-contracts/src/main/java/com/ecommerce/contracts/`
- Tests: `shared-contracts/src/test/java/com/ecommerce/contracts/`
- Generated sources (build output, not committed): `shared-contracts/target/generated-sources/`

---

## Phase 1: Setup (module + toolchain)

- [ ] T001 Create `shared-contracts/` Maven module with root `shared-contracts/pom.xml` (Java 17, `jar` packaging, `groupId` com.ecommerce, `artifactId` shared-contracts, version 1.0.0)
- [ ] T002 [P] Configure `protobuf-maven-plugin` (protoc + `protoc-gen-grpc-java`) bound to the `generate-sources` phase in `shared-contracts/pom.xml`
- [ ] T003 [P] Add dependencies to `shared-contracts/pom.xml`: `protobuf-java`, `grpc-protobuf`, `grpc-stub`, `jackson-databind`, `jakarta.validation-api`, `junit-jupiter` (test)
- [ ] T004 [P] Add `shared-contracts/buf.yaml` (lint rules over `src/main/proto`) and `shared-contracts/buf.gen.yaml`
- [ ] T005 [P] Configure JaCoCo plugin in `shared-contracts/pom.xml`: 80% line threshold, EXCLUDE generated proto sources (`**/com/ecommerce/contracts/**/v1/**` generated packages and `target/generated-sources/**`)
- [ ] T006 [P] Create source tree directories `shared-contracts/src/main/proto/{common,product,order}/` and `shared-contracts/src/main/java/com/ecommerce/contracts/{http,error}/`

**Checkpoint**: Empty module compiles (`mvn -pl shared-contracts compile`) and tooling is wired.

---

## Phase 2: Foundational (blocking prerequisites)

**Purpose**: Establish the shared error contract and prove the codegen pipeline works before
any story-specific contracts are added. MUST complete before Phase 3+.

- [ ] T007 Create `shared-contracts/src/main/proto/common/problem_details.proto` (RFC 7807 `ProblemDetail` proto mirror, package `ecommerce.common.v1`)
- [ ] T008 Verify codegen pipeline: `mvn -pl shared-contracts generate-sources` emits Java types from `problem_details.proto` into `target/generated-sources` (build smoke gate)

**Checkpoint**: Foundation ready — codegen confirmed; user-story phases can begin.

---

## Phase 3: User Story 1 — Consume contracts as a single versioned dependency (Priority: P1) 🎯 MVP

**Goal**: A service can declare `shared-contracts` at a pinned version and reference an HTTP
DTO, a gRPC message type, and a Kafka event type — all from the one artifact, with no locally
redefined contract types.

**Independent Test**: Reference a DTO + a proto message + an event type from the published
artifact and build successfully using only that dependency (SC-001, SC-002).

### Tests for US1 (write first — must fail before implementation)

- [ ] T009 [P] [US1] Write `ProblemDetailTest` (RFC 7807 JSON round-trip, `application/problem+json`, extension members preserved) in `shared-contracts/src/test/java/com/ecommerce/contracts/error/ProblemDetailTest.java`
- [ ] T010 [P] [US1] Write HTTP DTO round-trip tests for product + order DTOs in `shared-contracts/src/test/java/com/ecommerce/contracts/http/`
- [ ] T011 [P] [US1] Write `ContractSurfaceTest` asserting the artifact exposes the generated gRPC message types and event types (presence/sanity) in `shared-contracts/src/test/java/com/ecommerce/contracts/proto/ContractSurfaceTest.java`

### Implementation for US1

- [ ] T012 [P] [US1] Implement `ProblemDetail` HTTP DTO (RFC 7807, Jackson-annotated) in `shared-contracts/src/main/java/com/ecommerce/contracts/error/ProblemDetail.java`
- [ ] T013 [P] [US1] Create `shared-contracts/src/main/proto/product/product.proto` (`Product` message + `ProductService` gRPC contract, package `ecommerce.product.v1`)
- [ ] T014 [P] [US1] Create `shared-contracts/src/main/proto/order/order.proto` (`Order`, `OrderLine`, `OrderStatus` enum with the Principle V state values, `OrderService` gRPC contract, package `ecommerce.order.v1`)
- [ ] T015 [P] [US1] Implement product HTTP DTOs (with Bean Validation constraints) in `shared-contracts/src/main/java/com/ecommerce/contracts/http/product/`
- [ ] T016 [P] [US1] Implement order HTTP DTOs (with Bean Validation constraints) in `shared-contracts/src/main/java/com/ecommerce/contracts/http/order/`
- [ ] T017 [US1] Build & publish single versioned artifact: `mvn -pl shared-contracts clean install`; confirm one version (1.0.0) JAR in local `.m2` containing HTTP DTOs + generated gRPC + event types (FR-005, FR-007)

**Checkpoint**: US1 independently testable — all three contract categories consumable from one pinned artifact. **This is the MVP.**

---

## Phase 4: User Story 2 — Generate usable code from contract definitions (Priority: P2)

**Goal**: The build produces language-appropriate typed artifacts from the gRPC proto and the
Kafka event schemas, ready for producers/consumers/callers to use without hand-writing
serialization.

**Independent Test**: From a clean checkout, `mvn install` produces consumable typed
artifacts for proto + event definitions and they round-trip (US2 / FR-008).

### Tests for US2 (write first — must fail before implementation)

- [ ] T018 [P] [US2] Write generated-type round-trip test for `Product`/`Order` proto messages (serialize → parse equality) in `shared-contracts/src/test/java/com/ecommerce/contracts/proto/MessageRoundTripTest.java`
- [ ] T019 [P] [US2] Write event-type round-trip test for `StockMovementRecorded` + `OrderStateChanged` in `shared-contracts/src/test/java/com/ecommerce/contracts/proto/EventRoundTripTest.java`

### Implementation for US2

- [ ] T020 [P] [US2] Create `shared-contracts/src/main/proto/product/product_events.proto` (`StockMovementRecorded` append-only event + `StockMovementReason` enum, reusing `ecommerce.product.v1`)
- [ ] T021 [P] [US2] Create `shared-contracts/src/main/proto/order/order_events.proto` (`OrderStateChanged` event importing `order/order.proto`)
- [ ] T022 [US2] Confirm `mvn -pl shared-contracts generate-sources` emits typed gRPC stubs + event types for ALL proto and they package into the JAR (FR-008)

**Checkpoint**: US2 independently testable — codegen yields usable JVM types for gRPC + events.

---

## Phase 5: User Story 3 — Detect breaking changes through versioning (Priority: P3)

**Goal**: Backward-incompatible contract changes are flagged automatically and a clear
version + changelog signal is published.

**Independent Test**: Make an incompatible proto change, run the compatibility check, confirm
it fails and that a breaking release requires a major bump with a changelog entry (US3 /
FR-009a, SC-007).

### Tests for US3 (write first — must fail before implementation)

- [ ] T023 [P] [US3] Add a negative-check fixture/script proving `buf breaking` FAILS on an undeclared incompatible proto change (e.g., field renumber) in `shared-contracts/buf-breaking-negative-check` (test harness)
- [ ] T024 [P] [US3] Add a check asserting a backward-compatible additive change (new optional field) PASSES `buf breaking` (positive case)

### Implementation for US3

- [ ] T025 [US3] Add CI workflow `/.github/workflows/shared-contracts.yml` (or repo CI equivalent) running `buf lint` + `buf breaking --against main` and the JaCoCo 80% gate; build fails on undeclared breaking change (FR-009a, SC-006, SC-007)
- [ ] T026 [P] [US3] Create `shared-contracts/CHANGELOG.md` + a documented semver bump policy (breaking→major, additive→minor/patch; single version covers all categories) (FR-007, FR-009, FR-010, FR-011)
- [ ] T027 [US3] Create `shared-contracts/api-contract.md` documenting HTTP endpoints, gRPC services, and Kafka event contracts as governed by shared-contracts versioning (FR-013, constitution Principle II)

**Checkpoint**: US3 independently testable — CI compatibility gate + versioning/changelog in place.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T028 [P] Verify JaCoCo line coverage ≥ 80% on hand-written code; add tests for any gaps in `shared-contracts/src/test/java/`
- [ ] T029 [P] Document the N/N-1 interoperability policy (FR-010a) in `shared-contracts/api-contract.md` and `shared-contracts/README.md`
- [ ] T030 [P] Add `shared-contracts/README.md` quickstart (build, `mvn install`, consume as dependency) mirroring `specs/001-shared-contracts-schemas/quickstart.md`
- [ ] T031 Run full acceptance pass: `mvn -pl shared-contracts clean install` + `buf lint` + `buf breaking`; verify every smoke check in `quickstart.md` passes (FR-014, SC-001..SC-007)

---

## Dependencies & Execution Order

- **Setup (Phase 1)** → blocks everything.
- **Foundational (Phase 2)** → blocks all user stories (shared error proto + codegen proof).
- **US1 (Phase 3, P1)** → the MVP; depends only on Foundational.
- **US2 (Phase 4, P2)** → depends on Foundational; event protos build on US1 domain protos (T020/T021 reference `product.proto`/`order.proto` from T013/T014).
- **US3 (Phase 5, P3)** → depends on having proto to check (Foundational + at least US1 protos); fully effective once US1+US2 protos exist.
- **Polish (Phase 6)** → after all stories.

Story completion order: **US1 → US2 → US3** (priority order). US1 alone is a shippable MVP.

## Parallel Execution Examples

- **Setup**: T002, T003, T004, T005, T006 can run in parallel after T001 (distinct config/dirs).
- **US1 tests**: T009, T010, T011 in parallel (different test files).
- **US1 impl**: T012, T013, T014, T015, T016 in parallel (different files), then T017 (build) serially.
- **US2**: T018, T019 (tests) in parallel; T020, T021 (proto) in parallel; then T022.
- **US3**: T023, T024 in parallel; T026 parallel with T025; T027 after.

## Implementation Strategy

- **MVP = Phase 1 + Phase 2 + Phase 3 (US1)** — a published, versioned artifact whose HTTP
  DTOs, gRPC messages, and event types are consumable from one dependency. Ship this first.
- **Increment 2 = US2** — guarantees codegen produces usable typed artifacts for gRPC + events.
- **Increment 3 = US3** — adds the automated compatibility gate, changelog, and api-contract.md.
- TDD throughout: for every contract, its test task is listed and executed before its
  implementation task; generated proto sources are excluded from the coverage denominator.
