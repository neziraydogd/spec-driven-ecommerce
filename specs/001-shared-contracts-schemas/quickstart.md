# Quickstart: Shared-Contracts Schema Source of Truth

**Feature**: `001-shared-contracts-schemas` | **Date**: 2026-06-07

This quickstart shows how `shared-contracts` is built, published, and consumed once
implemented. Scope is **define-and-publish only** — no transport runtime, no Schema Registry.

## Prerequisites

- Java 17, Maven
- `buf` CLI installed (for lint + breaking-change checks)
- `protoc` is provided by `protobuf-maven-plugin` (no manual install needed)

## Build & publish locally

```bash
cd shared-contracts
mvn clean install            # generates Java types from proto, runs tests, installs to ~/.m2
```

This:
1. Runs `protobuf-maven-plugin` during `generate-sources` → Java message + gRPC stub types.
2. Compiles hand-written HTTP DTOs (incl. `ProblemDetail`).
3. Runs JUnit tests (DTO/RFC 7807 round-trips; generated-type sanity) + JaCoCo coverage.
4. Installs the single versioned artifact to the local Maven repository.

## Check contracts (CI gates, runnable locally)

```bash
cd shared-contracts
buf lint                     # style/structure rules over all .proto
buf breaking --against '.git#branch=main'   # fail on undeclared incompatible change
```

A breaking change without a matching major version bump fails CI (FR-009a, SC-007).

## Consume from a service

In `product-service/pom.xml` or `order-service/pom.xml`:

```xml
<dependency>
  <groupId>com.ecommerce</groupId>
  <artifactId>shared-contracts</artifactId>
  <version>1.0.0</version>   <!-- pinned; never a range -->
</dependency>
```

Then reference contracts directly — no local redefinition (SC-002):

```java
import com.ecommerce.contracts.error.ProblemDetail;          // HTTP DTO
import com.ecommerce.contracts.product.v1.Product;            // gRPC message type
import com.ecommerce.contracts.product.v1.StockMovementRecorded; // Kafka event payload
```

## Acceptance smoke checks (map to spec)

| Check | Validates |
|-------|-----------|
| Service builds referencing an HTTP DTO, a gRPC message, and an event type from the artifact | US1 / SC-001, SC-002 |
| `mvn install` produces generated typed artifacts from proto | US2 / FR-008 |
| HTTP DTO + ProblemDetail round-trip preserves all fields | US2 / FR-003, FR-004 |
| `buf breaking` flags an undeclared incompatible change and fails | US3 / FR-009a, SC-007 |
| Release bumps a single version + adds a changelog entry | US3 / FR-007, FR-011 |
| Invalid proto fails the build before publish | FR-014, SC-006 |

## Versioning rules (recap)

- One semantic version covers HTTP + gRPC + Kafka contracts together.
- Breaking → MAJOR; compatible additive → MINOR/PATCH.
- Consumers interoperate across N and N-1 during rollout.
- Every release adds a changelog entry and updates `api-contract.md`.
