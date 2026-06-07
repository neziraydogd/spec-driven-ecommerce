# E-Commerce Project Setup

## Prerequisites

Verify that the following tools are installed:

```
node -v
npm.cmd -v
ng --version
specify --version
claude --version
uv --version
```

### Installations

#### Node.js & NPM

Used for Angular development and package management.

* https://nodejs.org/en/download

#### Angular CLI

Used to create, build, and manage Angular applications.

```
npm.cmd install -g @angular/cli
```

#### Claude CLI

Used as the AI agent integration for Spec Kit.

```powershell
irm https://claude.ai/install.ps1 | iex
```

#### UV

Used to install and manage Python-based tooling.

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

#### Spec Kit CLI

Used for specification-driven development workflow.

```
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@v0.9.2
```

Alternative:

```
pip install specify-cli
```

---

## Initialization

```
cd /project
specify init --here
```

Select:

```text
y
claude
sh bash
```

Since Bash is selected, continue using Bash for all subsequent commands.

---

# 1. Create Constitution
## 1. Project Constitution

The following is a single Spec Kit command that initializes the project constitution:

```text

/speckit-constitution

E-commerce Platform — Project Constitution

# Technology
- Spring Boot 3.2, Java 17
- Monorepo with independent microservices (each service builds and deploys independently)
- Angular 17 frontend (standalone components, HttpClient for API layer)

# Repository Structure
Services in same repo, each with its own pom.xml:
  ecommerce-platform/
  ├── api-gateway/        (independent Spring Boot app)
  ├── product-service/    (independent Spring Boot app)
  ├── order-service/      (independent Spring Boot app)
  ├── frontend/           (Angular 17 app)
  ├── shared-contracts/   (published via local mvn install during dev)
  └── specs/              (cross-service Spec Kit files)

# shared-contracts
- Distributed as a library (mvn install to local .m2 for development)
- Contains: DTOs, API interfaces, error response models
- Services reference it as a Maven dependency, NOT via direct import
- Version bumped on every breaking change

# Architecture
- Services communicate via HTTP REST through api-gateway only
- Frontend never calls services directly — only api-gateway
- Each service owns its own database schema; no cross-service joins
- No shared runtime state between services
- Stock movements are immutable (append-only)
- Order state machine:
    DRAFT → CONFIRMED → SHIPPED → DELIVERED
    DRAFT → CANCELLED
    DELIVERED → RETURN_REQUESTED → RETURNED

# API Contract
- All endpoints documented in shared-contracts/api-contract.md
- Error responses follow RFC 7807 Problem Details format
- Breaking changes require version bump and api-contract.md update

# Cross-Cutting
- Auth: JWT-based, validated at api-gateway level only
- Database migrations: Flyway, per-service schema
- Global rules defined in CLAUDE.md (repo root)
- Agent-specific rules defined in AGENTS.md (repo root)
- Cross-service specs live in specs/ directory

# Engineering
- Each service buildable with: cd <service> && mvn clean install
- Tests written before implementation (TDD)
- Minimum 80% line coverage per service (enforced in CI)
- All CI checks must pass before merge
- Commit messages follow Conventional Commits
- Documentation and code comments in English
```
# Workflow

```text
/speckit-constitution   → done once
        ↓
/speckit-specify        → repeated for every new feature
        ↓
/speckit-plan           → plan generation for the specification
        ↓
/speckit-tasks          → implementation tasks
        ↓
Implementation
        ↓
New feature → start again from /speckit-specify
```
## 2. Specifying a Feature (/speckit-specify)
```bash
/speckit-specify
Developer can build product-service and order-service against a shared-contracts library that provides gRPC protos, Kafka event schemas, and HTTP DTOs for api-gateway — ensuring all inter-service contracts have a single versioned source of truth.
```
## 3. Clarify Spec

```bash
/speckit-clarify 
```
## 4. After Clarify,constitution resolve the gRPC/Kafka governance gap

## Amend the constitution to sanction gRPC and Kafka as inter-service
communication mechanisms, alongside HTTP REST. This is a MINOR version
bump (1.0.0 → 1.1.0) since it expands existing principles, not removes them.

Specifically:

1. Principle IV (Gateway-Mediated Communication):
    - Keep the rule that the FRONTEND calls api-gateway only (never services directly).
    - Keep JWT validation at api-gateway only.
    - BUT allow service-to-service communication over three sanctioned channels:
      synchronous HTTP REST, synchronous gRPC, and asynchronous Kafka events.
    - External/frontend-facing traffic still enters exclusively through api-gateway.

2. Principle I (Service Independence & Bounded Data Ownership):
    - Clarify that Kafka events are an allowed coordination mechanism and do NOT
      count as "shared runtime state." Services still own their schemas and must
      not share databases or in-memory state.

3. Technology & Repository Standards / shared-contracts description:
    - Update the shared-contracts contents to include: HTTP DTOs, gRPC proto
      definitions, Kafka event schemas, API interfaces, and the RFC 7807 error
      response model — as the single versioned source of truth for all contracts.

Update api-contract.md references to note that gRPC and Kafka contracts are also
governed by shared-contracts versioning.