# Phase 0 Research: Shared-Contracts Schema Source of Truth

**Feature**: `001-shared-contracts-schemas` | **Date**: 2026-06-07

All Technical Context items were resolved from the feature spec, the clarifications
session, the v1.1.0 constitution, and the user-provided technical direction. No
`NEEDS CLARIFICATION` markers remain.

## Decision 1: Single IDL — Protobuf (proto3) for both gRPC and Kafka events

- **Decision**: Express gRPC contracts *and* Kafka event schemas in proto3, in one
  `src/main/proto/` tree. Kafka event payloads reuse/compose the same message types used by
  gRPC where the domain concept is shared.
- **Rationale**: One schema language and one toolchain eliminates the cost of maintaining a
  second IDL (e.g., Avro). `shared-contracts` already needs proto for gRPC, so events come
  for free. Keeps the "single source of truth" promise literal — one definition per concept.
- **Alternatives considered**:
  - *Avro + Confluent Schema Registry* — the conventional Kafka default, but introduces a
    second schema language and a **runtime** registry service. Rejected: contradicts the
    define-and-publish-only scope (Q1=A) and adds operational weight this feature explicitly
    excludes.
  - *JSON Schema for events* — human-readable but a third format with weaker codegen and
    weaker breaking-change tooling for the JVM. Rejected: fragmentation.

## Decision 2: Code generation via protobuf-maven-plugin

- **Decision**: Use `protobuf-maven-plugin` to run `protoc` (with `protoc-gen-grpc-java`)
  during the Maven `generate-sources` phase, emitting Java message and gRPC stub types into
  `target/generated-sources`.
- **Rationale**: Native Maven integration (constitution mandates `mvn clean install` per
  module); generated sources are produced deterministically as part of the normal build and
  packaged into the published artifact. gRPC stub *types* are generated without pulling in a
  running server/client.
- **Alternatives considered**:
  - *`buf generate` as the primary generator* — capable, but splits the build away from
    Maven and complicates the `mvn install` flow. Kept `buf` for lint/breaking only.
  - *Checked-in generated code* — rejected: stale-artifact risk; generation must be
    reproducible from the IDL.

## Decision 3: Breaking-change detection with buf (CI gate)

- **Decision**: Run `buf lint` and `buf breaking` over the entire `.proto` tree in CI on
  every change. `buf breaking` compares against the last published baseline; an undeclared
  incompatible change fails the build. The required `shared-contracts` version bump severity
  is matched to the detected change (breaking → major; compatible additive → minor/patch).
- **Rationale**: Satisfies FR-009a (automated CI gate that fails on undeclared breaking
  changes) and SC-007 with a single tool covering both gRPC and Kafka proto, since both are
  proto3. Build-time enforcement aligns with define-and-publish scope (no runtime registry).
- **Alternatives considered**:
  - *Schema Registry compatibility checks* — runtime, rejected per scope.
  - *Manual review only* — cannot guarantee the 100% targets (SC-004/SC-007). Rejected.

## Decision 4: No runtime Schema Registry; define-and-publish only

- **Decision**: No Schema Registry, no transport wiring. The artifact ships proto IDL,
  generated JVM types, hand-written HTTP DTOs, the RFC 7807 model, and an `api-contract.md`.
- **Rationale**: Directly encodes clarification Q1=A and the user directive. Runtime schema
  enforcement is deferred to a future per-service transport-adoption feature; proto
  definitions will already exist, so that work starts unblocked.
- **Trade-off accepted**: Compatibility is enforced at CI/build time (buf), not at runtime.
  Acceptable because runtime event production/consumption is out of scope and the spec
  defines compatibility via a CI gate (Q3=B).

## Decision 5: JVM-only artifacts; HTTP DTOs hand-written with Jackson + Bean Validation

- **Decision**: Generate Java/JVM artifacts only (FR-008, Q2=A). HTTP DTOs are hand-written
  POJOs (Jackson-annotated, Jakarta Bean Validation constraints); `ProblemDetail` implements
  RFC 7807 for HTTP. gRPC/event surfaces use the generated proto types.
- **Rationale**: HTTP DTOs benefit from idiomatic Java validation/serialization and are the
  frontend/REST surface; proto types serve gRPC and events. No TypeScript generation
  (frontend type generation is out of scope, Q2=A).
- **Alternatives considered**:
  - *Generate HTTP DTOs from proto too* — possible, but proto-to-REST mapping adds friction
    (JSON naming, validation) for little gain at this scope. Rejected for now; revisitable.

## Decision 6: Single shared version across all categories; N/N-1 compatibility

- **Decision**: One semantic version on the `shared-contracts` artifact covers HTTP, gRPC,
  and Kafka contracts together (FR-007). Consumers on the latest version must interoperate
  with peers one compatible version behind (N and N-1, FR-010a). Each release carries a
  changelog (FR-011).
- **Rationale**: A single version is simpler to reason about for independently deployed
  services and matches the constitution's versioning policy. N/N-1 covers the realistic
  rolling-deploy gap without forcing indefinite backward support.
- **Alternatives considered**: *Per-category versions* — rejected (Q-clarified single
  version); *whole-major-line compatibility* — rejected as over-broad (Q4=B).

## Coverage / testing note

- protoc-generated sources are excluded from the JaCoCo coverage denominator; the 80% floor
  (Principle III) applies to hand-written code (HTTP DTOs, RFC 7807 model, any helpers).
- Tests authored before implementation per Principle III (TDD).
