---
name: architecture
description: >
  Apply system architecture standards across all backend, frontend, and mobile stacks.
  Use this skill when making structural decisions: folder layout, layering, dependency direction,
  API design, database modeling, state management, state classification, cache strategy,
  or cross-cutting concerns.
---

# Architecture Skill

## Overview

Architecture decisions are the hardest to reverse. This skill defines the structural standards that apply across all projects — regardless of stack. The principles are stack-agnostic first, then stack-specific.

**Guiding principles:**
- Feature/domain-first structure, never type-first
- Dependency flows inward only (never domain → infrastructure)
- Boundaries are explicit; internals are private
- Stateless processes; externalize all state

---

## 1. Universal Architecture Rules

These rules apply to **every project**, regardless of language or framework.

### 1.1 Folder Structure — Feature First

```
PREFER this (feature-first):
  src/
    order/
      api/         ← HTTP handlers / controllers
      application/ ← use cases, orchestration
      domain/      ← entities, value objects, business rules
      infrastructure/ ← DB, external APIs, message queues
    billing/
    user/

AVOID this (type-first):
  src/
    controllers/
    services/
    repositories/
    utils/
```

### 1.2 Dependency Direction

Dependencies must flow **inward only**:

```
Interface (HTTP / UI / CLI)
    ↓
Application (use cases / orchestration)
    ↓
Domain (entities / business rules) ← Infrastructure (DB / external APIs)
```

**Rules:**
- Domain layer MUST NOT depend on frameworks, ORMs, or external SDKs
- Infrastructure adapts to domain interfaces (implements ports/interfaces defined in domain)
- Application layer coordinates domain and infrastructure — never skips to infrastructure directly

### 1.3 Stateless Processes

- No sticky sessions
- No in-process mutable shared state as source of truth
- No local disk as persistent storage
- Dev / staging / production use the **same version** of backing services
- Observability built in by default: logs, metrics, traces

### 1.4 DTO / Domain / Persistence Separation

```
Request DTO   → Application boundary input (validated, typed)
Domain Model  → Business rules entity (no framework annotations)
Response DTO  → Output shape (caller-friendly, versioned)
ORM Entity    → Database persistence concern only

NEVER: expose ORM entity directly as API response
NEVER: use API request DTO inside domain logic
```

### 1.5 Transaction Boundaries

- Transactions belong in the **application layer** (use case / service)
- Never place `@Transactional` / transaction logic in: controllers, entities, event listeners
- One method = one transaction = one bounded operation
- Forbidden: a single method that silently does DB write + external API call + event publish

### 1.6 Validation Placement

```
Boundary (Controller / Handler):
  → Input format validation (type, required, length, format)
  → Returns user-friendly error messages (HTTP 400)

Domain / Application:
  → Business invariants (e.g., "order total must be positive")
  → Throws domain exceptions on invariant violation

Database:
  → Hard constraints (NOT NULL, UNIQUE, CHECK)
  → Last line of defense — DB constraints enforce what app validation might miss
```

---

## 2. Backend Architecture Standards

### 2.1 Spring Boot

**Package structure:**
```
com.acme.<domain>.api           ← controllers, request/response DTOs
com.acme.<domain>.application   ← use cases, application services
com.acme.<domain>.domain        ← entities, value objects, domain services
com.acme.<domain>.infrastructure← JPA, REST clients, messaging
```

**Rules:**
- Controller: request binding + auth context extraction + response mapping only
- Business rules and transactions: application service / use case only
- Transaction boundaries: application service only — not in controller, entity, or listener
- Never serialize JPA entity directly to API response
- Config values: typed `@ConfigurationProperties` + environment variables / secret manager
- **Forbidden:** default package, controller → repository direct call, shared `util` package

### 2.2 NestJS

**Module structure:**
```
src/modules/<feature>/
  presentation/   ← controllers, DTOs, pipes
  application/    ← use cases, orchestration
  domain/         ← entities, value objects
  infrastructure/ ← TypeORM, HTTP clients, providers
```

**Rules:**
- Guard: authentication & authorization checks
- Interceptor: logging, metrics, serialization
- Pipe: input validation and transformation
- Exception Filter: maps errors to HTTP responses
- Each feature module exports only its public API (no cross-feature internals)
- **Forbidden:** giant `common/` dumping ground, query builder in controller, globally mutable singletons

### 2.3 Python

**Structure:**
```
src/
  domain/         ← entities, value objects, domain services
  application/    ← use cases
  infrastructure/ ← DB, HTTP, message queue adapters
  interfaces/     ← CLI, HTTP handlers, event consumers
```

**Rules:**
- All public functions/classes/modules must have type hints
- No import-time side effects (no network calls, DB connections, or env initialization on `import`)
- Exceptions must be explicit and consistent — no mixed `None | dict | False` returns
- Public APIs require docstrings
- **Forbidden:** giant `utils.py`, untyped dict pass-through chains, hidden side-effect module imports

### 2.4 Go

**Structure:**
```
internal/               ← all application logic
  <domain>/
    handler.go          ← HTTP handlers
    service.go          ← business logic
    repository.go       ← data access interface
cmd/<app>/main.go       ← entry point only
```

**Rules:**
- `gofmt` / `goimports` mandatory — style debate forbidden
- Package names: short, lowercase, meaningful. Forbidden: `util`, `common`, `types`, `api`
- Normal flow control: use error returns. Never panic for business failures.
- No `Get` prefix on getters
- **Forbidden:** exporting packages without reason, giant shared packages, swallowing errors with log-only

### 2.5 Rust

**Structure:**
```
crates/
  <domain>/        ← library crate: domain + application core
  <app>/           ← binary crate: wiring + entry point
```

**Rules:**
- Multi-app → Cargo workspace
- Recoverable errors: `Result<T, E>`. `panic!` only for invariant bugs or unrecoverable bootstrap.
- Traits as ports/interfaces; adapters implement traits
- `rustfmt` and `clippy` mandatory
- **Forbidden:** `unwrap()`/`expect()` in production paths without audit, unsupported `unsafe`, single-giant-module antipattern

---

## 3. Frontend Architecture Standards

### 3.1 React

**Structure:**
```
src/
  features/<feature>/
    ui/       ← React components, JSX
    model/    ← business logic, selectors, reducers
    api/      ← data fetching, server state
    lib/      ← pure utilities for this feature
  shared/
    ui/       ← shared components (Button, Input, Modal)
    lib/      ← shared utilities
  app/        ← routing, providers, entry
```

**Rules:**
- Components and hooks must be **pure and predictable** — no side effects during render
- Hooks: always called at top level, never conditionally
- Minimize stored state — derive what you can from existing state
- **Never mix** server state / UI state / form state / URL state
- Global state only for genuinely cross-route shared data
- JSX contains rendering logic only — move auth rules, date logic, price calculations to `model/lib`
- **Forbidden:** leaf component initiating network fetches, cross-feature internal file imports

### 3.2 Next.js (App Router)

**Structure:**
```
app/              ← routing, layouts, compositions only
features/         ← domain logic, per-feature modules
server/           ← server-only services, DB access
shared/           ← shared components, utilities
```

**Rules:**
- Default: **Server Component**. Add `use client` only at the smallest interactive leaf.
- Secrets, DB access, internal API calls → server components / route handlers / `server-only` modules only
- Route Handlers: HTTP adapter only (validation + auth context + response mapping). Delegate business logic to use cases.
- Cache/revalidation policy must be explicit — never rely on defaults silently.
- **Forbidden:** making full pages client components by habit, secret/DB code in client bundle, business logic accumulation in route files

---

## 4. Mobile Architecture Standards

### 4.1 Android

**Architecture layers:**
```
UI Layer
  ├── Activity / Fragment / Composable  ← render state, forward intents
  └── ViewModel                         ← state owner, exposes StateFlow/UiState

Domain Layer (optional, use when reuse or complexity warrants it)
  └── UseCase / Interactor

Data Layer
  ├── Repository                        ← combines local + remote sources
  ├── Local DataSource (Room/DataStore)
  └── Remote DataSource (Retrofit)
```

**Rules:**
- UI components: render state + forward user intent only. No business logic.
- ViewModel: single source of truth for UI state. Survives config changes.
- Repository: hides Room/Retrofit/DataStore details from upper layers.
- Layer communication: unidirectional data flow using coroutines/Flow.
- Multi-module: split by feature when the app grows.
- **Forbidden:** UI directly calling data source, business logic in UI, god ViewModel, all code in single app module

### 4.2 iOS

**Architecture pattern:**
```
View (SwiftUI)     ← declarative, lightweight
    ↓
ViewModel / Owner  ← state + user intent handling
    ↓
Repository / Service ← data and side effects
```

**Rules:**
- SwiftUI-first for all new UI. UIKit only for legacy screens or framework gaps.
- One source of truth per screen/flow. State owner and view must be clearly separated.
- Views: declarative only. Orchestration, side effects, navigation decisions → separate owner.
- Structured concurrency by default. Strictly limit mutable shared state across concurrency boundaries.
- Public API names must read naturally at the call site.
- **Forbidden:** networking/persistence inside View/ViewController, multiple state sources for same screen, shared concurrency objects without safety audit

---

## 5. Database Architecture (DDD-Aligned)

### Ownership Rules
- Each bounded context (BC) owns its own DB schema
- Cross-BC direct schema writes are **forbidden**
- Cross-BC reads via: API calls, published events, or dedicated read models only

### Naming Conventions
```
Schema name:   bounded context name
Table name:    singular snake_case (e.g., order, billing_item)
Primary key:   id
Foreign key:   <target>_id (e.g., user_id, order_id)
```

### Aggregate Rules
- Repository per aggregate root only
- Write transaction boundary = one aggregate command

### CQRS Guidance
- Write model: optimized for invariants and consistency
- Read model: denormalization allowed for query performance
- Simple CRUD domains do NOT require full CQRS/DDD — apply proportionally to complexity

### Cross-Boundary Rules
```
FORBIDDEN:
  - Cross-BC JOIN queries
  - Cross-BC foreign keys
  - Shared write tables between BCs

ALLOWED alternatives:
  - API calls to the owning BC
  - Subscribing to integration events
  - Materialized read models per consumer
```

### Event Rules
```
Domain event:     internal to the BC (triggers internal reactions)
Integration event: published AFTER successful commit (external propagation)
Reliability:      outbox table pattern for at-least-once delivery guarantee
```

### Migration Rules
- All schema changes via versioned migration files only (Flyway, Liquibase, Alembic, etc.)
- Breaking changes (column removal, type narrowing) → two-phase deployment

### Audit Fields
```
created_at  — mandatory on all tables
updated_at  — mandatory on all tables
deleted_at  — optional, only if soft delete is required for recovery/audit
```

---

## 6. API Design Standards (OpenAPI / Swagger / Scalar)

### Contract-First Principle
The `openapi.yaml` or generated `openapi.json` in the repository is the **single source of truth** for the API contract. Implementation follows the spec; the spec does not follow the implementation.

### Minimum Documentation per Endpoint
Every endpoint must have:
```
- summary
- operationId
- tags
- security (auth scheme)
- path / query / header parameters
- request body schema
- response schema (2xx, 4xx, 5xx)
- example request and response payloads
```

### CI Validation Rules
```
[ ] OpenAPI spec validation passes on every PR
[ ] No undocumented routes
[ ] No example/schema mismatch
[ ] No breaking changes without version/policy review
```

### Breaking Change Definition
Any of the following requires a version increment or deprecation notice:
- Removing a response field
- Adding a required request field
- Narrowing an enum
- Changing authentication scheme

### Security Rules
- Internal / admin API Swagger endpoints must NOT be publicly deployed
- If exposed, they must be behind authentication middleware

### Stack-Specific Integration
```
Spring Boot:  springdoc-openapi → generates spec + Swagger UI
NestJS:       @nestjs/swagger → OpenAPI doc + @scalar/nestjs-api-reference for reference page
Next.js:      serve static openapi.json + @scalar/nextjs-api-reference
Design-first: Swagger Editor → code generation pipeline
```

---

## 7. Architecture Decision Records (ADR)

Write an ADR for every significant structural decision:

```markdown
# ADR-XXXX: [Decision Title]

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-XXXX

## Context
What situation or problem prompted this decision?

## Decision
What was decided?

## Rationale
Why was this option chosen over alternatives?

## Consequences
What becomes easier? What becomes harder? What are the trade-offs?

## Alternatives Considered
What other options were evaluated?
```

ADRs live in `architecture/decisions/` and are **never deleted** — only marked Deprecated or Superseded.

---

## 8. State Management Standards

Detailed rules are in [`ref/state-management.md`](ref/state-management.md). Key principles below.

### State Classification

| Category | Scope | Owner |
|---|---|---|
| Ephemeral UI State | Component-local (hover, focus, scroll position) | Component/View/Widget |
| Screen State | Per-screen (list data, loading/error, pagination) | ViewModel / Controller |
| App State | App-wide (auth, permissions, theme) | App root / feature boundary |
| Server State | Cached remote data | Cache layer (React Query, SWR, etc.) |
| Backend Authority State | Authoritative business data (orders, payments) | DB / event store / distributed cache |

### Core Rules

- **Minimize stored state** — derive what you can from source state + functions.
- **Single source of truth** — DO NOT store the same semantic state in two places.
- **Subscription scope** — high-frequency state MUST be subscribed by the smallest unit. FORBIDDEN: entire screen subscribing to global state.
- **Async state** — every request must be cancellable. Separate loading/error/empty/success. Prevent stale responses from overwriting current state.
- **List state** — manage by page/cursor/visible window. DO NOT hold entire datasets in memory.

### Backend State Rules

- App servers are **stateless / share-nothing** by default.
- DO NOT store authoritative state in process memory.
- Cache is NOT the authoritative source — every entry MUST have TTL or explicit invalidation.
- With multiple instances, prefer **distributed cache** over local in-memory cache.
- Document which state has which lifecycle (request / session / workflow / cache / durable).

---

## 9. Architecture Checklist

Before finalizing any structural decision:

```
[ ] Feature/domain-first folder structure applied
[ ] Dependency direction is inward only (no domain → infra)
[ ] DTO / Domain Model / ORM Entity / Response DTO separated
[ ] Transaction boundaries placed in application layer only
[ ] Business logic not leaking into controllers, entities, or UI
[ ] Validation at boundary; invariants in domain
[ ] Stateless process: no local disk persistence, no sticky sessions
[ ] Cross-BC communication via API or events (no shared DB writes)
[ ] All schema changes via versioned migrations
[ ] OpenAPI spec is source of truth and validated in CI
[ ] Breaking API changes flagged and versioned
[ ] ADR written for significant decisions
[ ] State classification documented (ephemeral / screen / app / server / backend authority)
[ ] Single source of truth for each piece of state
[ ] Cache entries have TTL or explicit invalidation strategy
[ ] Backend processes are stateless — authority state in external stores
[ ] High-frequency state subscription scope is minimal
```
