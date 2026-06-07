<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 1.1.0
Rationale: MINOR amendment — expands two existing principles and the shared-contracts
definition to sanction gRPC and Kafka as inter-service communication mechanisms alongside
HTTP REST. No principle removed or redefined incompatibly, so MINOR per the versioning
policy.

Modified principles:
- I. Service Independence & Bounded Data Ownership — clarified that Kafka events are an
  allowed coordination mechanism and do NOT constitute "shared runtime state."
- IV. Gateway-Mediated Communication — service-to-service traffic may now use HTTP REST,
  gRPC, or Kafka events; frontend/external ingress remains api-gateway-only; JWT still
  validated at api-gateway only.
- II. Contract-First Integration — shared-contracts now also holds gRPC proto definitions
  and Kafka event schemas; their breaking changes are governed by the same versioning rule.

Added sections: none (existing sections expanded)

Removed sections: none

Templates requiring updates:
- ✅ .specify/templates/plan-template.md — Constitution Check gate references remain
  generic ("Gates determined based on constitution file"); no edit needed.
- ✅ .specify/templates/spec-template.md — no constitution-coupled mandatory sections
  changed; no edit needed.
- ✅ .specify/templates/tasks-template.md — no structural change; TDD (III) still applies.
- ⚠ README.md / CLAUDE.md / AGENTS.md — should reference this constitution and the
  expanded communication channels (manual follow-up, non-blocking).
- ⚠ shared-contracts/api-contract.md — must document gRPC and Kafka contracts as governed
  by shared-contracts versioning (created during implementation; pending).

Follow-up TODOs: none. RATIFICATION_DATE unchanged (2026-06-07); LAST_AMENDED_DATE bumped.
-->

# E-commerce Platform Constitution

## Core Principles

### I. Service Independence & Bounded Data Ownership

Each microservice (`api-gateway`, `product-service`, `order-service`) MUST build,
test, and deploy independently with its own `pom.xml` and its own database schema.
Cross-service database joins are PROHIBITED; a service MUST NOT read or write another
service's schema. No shared runtime state is permitted between services — coordination
happens exclusively through published contracts and the sanctioned communication channels
of Principle IV. Asynchronous Kafka events are an allowed coordination mechanism and do
NOT constitute shared runtime state: each service still owns its schema and MUST NOT share
a database or in-memory state with another service. Database migrations MUST be managed
with Flyway, scoped per-service.

**Rationale**: Independent deployability and data ownership are the defining value of
microservices; sharing schemas or in-memory state reintroduces the coupling the
architecture exists to eliminate. Event-based coordination preserves that independence
because each service consumes events on its own terms into its own store.

### II. Contract-First Integration

All inter-service and frontend-facing contracts MUST live in `shared-contracts` and be
consumed as a Maven dependency via local `mvn install`. `shared-contracts` is the single
versioned source of truth for: HTTP DTOs, gRPC proto definitions, Kafka event schemas, API
interfaces, and the RFC 7807 Problem Details error response model. Direct source imports
across service modules are PROHIBITED. Every HTTP endpoint MUST be documented in
`shared-contracts/api-contract.md`, and gRPC and Kafka event contracts are governed by the
same `shared-contracts` versioning. Error responses MUST follow the RFC 7807 Problem
Details format. Any breaking change to any contract (HTTP, gRPC, or Kafka) MUST bump the
`shared-contracts` version AND update `api-contract.md` in the same change.

**Rationale**: A single versioned contract artifact is the only safe way for
independently deployed services to evolve without silent breakage.

### III. Test-First (NON-NEGOTIABLE)

Tests MUST be written before implementation. The Red-Green-Refactor cycle is mandatory:
tests are authored, observed to fail, then implementation makes them pass. Each service
MUST maintain a minimum of 80% line coverage, enforced in CI. A change that lowers
coverage below the threshold MUST NOT merge.

**Rationale**: TDD with an enforced coverage floor is the project's primary defense
against regressions in a system where services deploy on independent cadences.

### IV. Gateway-Mediated Communication

External and frontend-facing traffic MUST enter the platform exclusively through
`api-gateway`. The frontend MUST call `api-gateway` only and MUST NOT call any service
directly. Authentication is JWT-based and MUST be validated at the `api-gateway` level
only; downstream services trust the gateway-validated identity and MUST NOT re-implement
token validation.

Service-to-service communication MUST use one of three sanctioned channels and no others:
synchronous HTTP REST, synchronous gRPC, or asynchronous Kafka events. All three are
governed by contracts published in `shared-contracts` (Principle II). Services MUST NOT
communicate through any unsanctioned channel (e.g., direct database access or shared
in-memory state).

**Rationale**: A single external ingress and auth boundary keeps security and routing
concerns in one auditable place and prevents the frontend from coupling to internal
topology. Permitting gRPC and Kafka for internal traffic — while requiring every channel to
be contract-governed — lets services choose the right interaction style (request/response
vs. event-driven) without sacrificing the contract-first guarantee.

### V. Domain Integrity & Immutability

Stock movements MUST be append-only; existing movement records MUST NOT be mutated or
deleted. Order lifecycle transitions MUST follow the defined state machine and MUST
reject any transition not explicitly allowed:

- `DRAFT → CONFIRMED → SHIPPED → DELIVERED`
- `DRAFT → CANCELLED`
- `DELIVERED → RETURN_REQUESTED → RETURNED`

**Rationale**: An immutable stock ledger and an explicit state machine give the platform
an auditable, reconstructable history and prevent illegal business states.

## Technology & Repository Standards

- Backend services MUST use Spring Boot 3.2 on Java 17.
- The frontend MUST be Angular 17 using standalone components, with `HttpClient` as the
  API layer.
- The repository is a monorepo with the fixed layout: `api-gateway/`, `product-service/`,
  `order-service/`, `frontend/`, `shared-contracts/`, and `specs/`.
- `shared-contracts` holds HTTP DTOs, gRPC proto definitions, Kafka event schemas, API
  interfaces, and the RFC 7807 error response model — distributed as a versioned Maven
  library and serving as the single source of truth for all inter-service contracts.
- Cross-service Spec Kit files MUST live under `specs/`.
- Global repository rules are defined in `CLAUDE.md` (repo root); agent-specific rules in
  `AGENTS.md` (repo root). This constitution supersedes both where they conflict.

## Development Workflow & Quality Gates

- Each service MUST be buildable in isolation with `cd <service> && mvn clean install`.
- All CI checks (build, tests, 80% coverage gate) MUST pass before merge.
- Commit messages MUST follow Conventional Commits.
- All documentation and code comments MUST be written in English.
- Breaking API changes MUST NOT merge without the corresponding `shared-contracts`
  version bump and `api-contract.md` update (see Principle II).

## Governance

This constitution supersedes all other development practices. When `CLAUDE.md`,
`AGENTS.md`, or any template conflicts with this document, this document prevails.

Amendments MUST be proposed via pull request, documented in the Sync Impact Report at the
top of this file, and approved before merge. Versioning follows semantic versioning:

- **MAJOR**: Backward-incompatible governance changes — principle removals or
  redefinitions.
- **MINOR**: A new principle or section is added, or existing guidance is materially
  expanded.
- **PATCH**: Clarifications, wording, or non-semantic refinements.

All pull requests and reviews MUST verify compliance with these principles. Any deviation
or added complexity MUST be justified in the plan's Complexity Tracking section; an
unjustified violation blocks merge. Agents and contributors MUST use `CLAUDE.md` and
`AGENTS.md` for runtime development guidance, subject to the supremacy of this
constitution.

**Version**: 1.1.0 | **Ratified**: 2026-06-07 | **Last Amended**: 2026-06-07
