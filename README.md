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
```text
/speckit-specify
Developer can build product-service and order-service against a shared-contracts library that provides gRPC protos, Kafka event schemas, and HTTP DTOs for api-gateway — ensuring all inter-service contracts have a single versioned source of truth.
```

