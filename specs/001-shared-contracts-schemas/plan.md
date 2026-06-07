# Implementation Plan: Shared-Contracts Schema Source of Truth

**Branch**: `001-shared-contracts-schemas` | **Date**: 2026-06-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-shared-contracts-schemas/spec.md`

## Summary

Establish `shared-contracts` as the single, versioned source of truth for all
inter-service contracts in the platform: HTTP DTOs (with the RFC 7807 error model), gRPC
proto definitions, and Kafka event schemas. Kafka event schemas are expressed in Protobuf
(proto3), reusing the same IDL and toolchain as gRPC — one schema language, one build
pipeline. The build generates Java/JVM artifacts via `protobuf-maven-plugin` (protoc +
protoc-gen-grpc-java) and publishes a single versioned Maven artifact to the local `.m2`.
Backward-compatibility is enforced at CI time by `buf` (lint + breaking-change detection)
over all `.proto` files, with the version bump severity matched to the detected change.
Scope is **define-and-publish only**: no gRPC servers/clients, no Kafka producers/consumers,
and **no runtime Schema Registry** are part of this feature.

## Technical Context

**Language/Version**: Java 17 (per constitution); proto3 IDL for gRPC + Kafka schemas.

**Primary Dependencies**: Maven; `protobuf-maven-plugin` (drives `protoc` + the
`protoc-gen-grpc-java` plugin); `protobuf-java` runtime; `grpc-stub`/`grpc-protobuf` (stub
types only — no server/client runtime); `buf` CLI (lint + breaking detection in CI);
Jackson (HTTP DTO (de)serialization); Jakarta Bean Validation API (DTO constraint
annotations); JUnit 5 + JaCoCo (tests + coverage).

**Storage**: N/A — `shared-contracts` is a contract library; it owns no database and holds
no runtime state.

**Testing**: JUnit 5 unit tests (HTTP DTO + RFC 7807 serialization round-trips; generated
event/message type sanity); `buf lint` and `buf breaking` as CI gates over `.proto`;
JaCoCo line-coverage report. Coverage threshold applies to hand-written code only;
protoc-generated sources are excluded from the JaCoCo denominator.

**Target Platform**: JVM library artifact (JAR) installed to the local Maven repository
(`mvn install`) and consumed by the Spring Boot services as a versioned dependency.

**Project Type**: Maven library module within the existing monorepo (`shared-contracts/`).
No web/mobile app structure applies.

**Performance Goals**: N/A at runtime (build-time artifact). Non-functional target: a clean
`mvn clean install` of `shared-contracts` (including code generation) completes in well
under typical CI step budgets; `buf` checks run in seconds.

**Constraints**: Define-and-publish only — NO transport wiring (gRPC servers/clients, Kafka
producers/consumers) and NO runtime Schema Registry. Single shared version covers all
contract categories together. Interoperability guaranteed across N and N-1 (FR-010a).
JVM-only generated artifacts (no TypeScript/frontend generation).

**Scale/Scope**: Small, bounded — contract definitions covering the product and order
domains plus the shared RFC 7807 error model; a handful of proto files and DTO classes,
not a large codebase.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Constitution version **1.1.0**. Gate evaluation per principle:

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Service Independence & Bounded Data Ownership | ✅ PASS | `shared-contracts` is a library with no database, no schema, and no runtime state. It does not enable cross-service joins. Kafka schemas it defines are proto3 (allowed coordination mechanism, not shared runtime state). |
| II. Contract-First Integration | ✅ PASS (this feature *implements* it) | This is the canonical contracts module. v1.1.0 explicitly names HTTP DTOs, gRPC protos, Kafka event schemas, API interfaces, and the RFC 7807 model as its contents. Single versioned artifact; breaking changes bump version + update `api-contract.md`. |
| III. Test-First (NON-NEGOTIABLE) | ✅ PASS | Tests authored before implementation; 80% line coverage enforced over hand-written code (generated proto sources excluded from JaCoCo). |
| IV. Gateway-Mediated Communication | ✅ PASS | Defines contracts for the three sanctioned service-to-service channels (HTTP REST, gRPC, Kafka events). No transport runtime is wired here, so the gateway/ingress and JWT rules are untouched. |
| V. Domain Integrity & Immutability | ✅ PASS | Contracts model append-only stock movements and the order state machine; the library defines shapes only and cannot violate runtime invariants. |

**Result**: All gates PASS. No violations → Complexity Tracking left empty.

*Post-Phase-1 re-check*: Design (Maven module + proto layout + buf config + DTO classes)
introduces no new constitutional concerns. Gates still PASS.

## Project Structure

### Documentation (this feature)

```text
specs/001-shared-contracts-schemas/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (proto + DTO contract sketches)
└── tasks.md             # Phase 2 output (/speckit-tasks — NOT created here)
```

### Source Code (repository root)

```text
shared-contracts/
├── pom.xml                      # Maven module: protobuf-maven-plugin, protobuf-java,
│                                #   grpc stubs, jackson, jakarta.validation, junit, jacoco
├── buf.yaml                     # buf module config (lint rules)
├── buf.gen.yaml                 # buf generate config (optional; protoc via maven is primary)
├── src/
│   ├── main/
│   │   ├── proto/               # Single source of truth for proto3 IDL
│   │   │   ├── common/
│   │   │   │   └── problem_details.proto   # RFC 7807 (gRPC/event surface)
│   │   │   ├── product/
│   │   │   │   ├── product_messages.proto  # gRPC message types
│   │   │   │   ├── product_service.proto   # gRPC service defs
│   │   │   │   └── product_events.proto    # Kafka event payloads (proto3)
│   │   │   └── order/
│   │   │       ├── order_messages.proto
│   │   │       ├── order_service.proto
│   │   │       └── order_events.proto      # incl. stock movement + order state events
│   │   └── java/
│   │       └── com/ecommerce/contracts/
│   │           ├── http/        # Hand-written HTTP DTOs
│   │           │   ├── product/
│   │           │   └── order/
│   │           └── error/
│   │               └── ProblemDetail.java  # RFC 7807 HTTP DTO
│   └── test/
│       └── java/
│           └── com/ecommerce/contracts/
│               ├── http/        # DTO + RFC 7807 serialization round-trip tests
│               └── proto/       # generated-type sanity tests
└── api-contract.md              # Endpoint + event + gRPC contract documentation
```

**Structure Decision**: A single Maven library module `shared-contracts/` at the monorepo
root (matching the constitution's fixed layout). All proto3 IDL lives under
`src/main/proto/` as the one source of truth; `protobuf-maven-plugin` generates Java
message/stub types into `target/generated-sources` during `generate-sources`. Hand-written
HTTP DTOs live under `src/main/java`. `buf.yaml` governs lint/breaking checks over the
proto tree. No `src/main/java` transport code (servers, clients, producers, consumers) is
created — only types and definitions.

## Complexity Tracking

> No constitution violations. Section intentionally left empty.
