---
name: ddd-and-onion
description: Domain-Driven Design building blocks and Clean/Onion Architecture layer model. Load this skill when designing or implementing any feature. Defines ubiquitous language, all DDD tactical patterns, the 4-layer architecture with strict dependency rules, Independent Definitions Principle, explicit mapping strategy, encapsulation rules, and compliance verification checklist.
---

# Domain-Driven Design and Clean Architecture

DDD provides the *what* — how to model the domain. Clean/Onion Architecture provides the *where* — how to organize code into layers. The two practices are used together on every feature.

---

## Part 1 — Domain-Driven Design

### Ubiquitous Language

The single most important DDD practice. Every concept in the domain has one name, used consistently everywhere: domain experts, developers, tests, error messages, logs.

- If the domain calls it an `Order`, the code has `Order` — not `OrderRecord`, not `OrderEntity`, not `OrderModel`
- If the business says "place an order", the method is `PlaceOrder` — not `CreateOrder`, not `SubmitOrder`
- Inconsistent naming is a design smell: the model doesn't reflect the domain

When a concept has no clear domain name, resolve it before writing code. It is a design problem, not a naming convention decision.

### Bounded Context

A bounded context is the boundary within which a domain model applies. The same word can mean different things in different contexts.

- In the `Sales` context, a `Customer` has credit limit and purchase history
- In the `Shipping` context, a `Customer` has delivery address and preferences
- These are different models — they must not share a `Customer` class

**Define explicitly in every spec:**
- **Name**: what bounded context is this?
- **Owns**: which domain concepts live here?
- **Does not own**: what is another context's responsibility?

---

### DDD Building Blocks

#### Aggregate

A consistency boundary. A cluster of entities and value objects treated as a **single unit for changes**.

- **Aggregate Root** — the only entity external code may interact with
- Enforces all invariants (business rules) within its boundary
- References other aggregates by **ID only** — never by object reference
- Unit of persistence: load and save the entire aggregate
- One aggregate per transaction; use domain events for cross-aggregate coordination
- Keep aggregates small — include only what must change together

```
// Good: aggregate root controls state through its own methods
order.addItem(productId, quantity)

// Bad: reaching inside the aggregate from outside
order.items.push(new OrderItem(productId, quantity))
```

**Encapsulation (non-negotiable):**
- Private constructor — no external `new MyAggregate()`
- Static `create()` method or factory enforces validation before instantiation
- Readonly `_id` for immutable identity
- Private fields with controlled getters
- Domain methods (not raw setters) mutate state after validation
- Idempotent operations to prevent duplicate state

**Expose Only:**
- Root entity ID
- Getters for querying state
- Business behavior methods
- Identity equality (`equals()`)
- Factory/static `create()` methods

**Never expose:**
- `toJSON()`, `toDTO()`, `toPlainObject()` — serialization is a mapper's job, not the aggregate's

#### Entity

An object with a **unique identity** that persists across state changes. Two entities with the same ID are the same entity even if attributes differ.

**Use when:**
- Object has a meaningful lifecycle (created, modified, archived)
- Must be tracked and distinguished from similar objects

**Not when:**
- Fully described by its attributes → use Value Object

**Design:** enforce own invariants, use intention-revealing methods instead of raw setters. Make invalid states unrepresentable.

**Never expose:** `toJSON()`, `toDTO()` — use mappers at layer boundaries.

#### Value Object

An **immutable** object defined entirely by its attributes. No identity. Equality by value.

```
// Value equality: same attributes = same value object
Email("user@example.com") == Email("user@example.com")  // true
```

**Use for:** measurements (Money, Quantity), descriptions (Address, Email), domain identifiers that carry validation rules. Prefer value objects over primitives for domain concepts.

**Design:**
- All fields required and validated at construction
- Operations return new instances — never mutate
- No serialization methods — use mappers

**Never expose:** `toJSON()`, `toDTO()` — use mappers.

#### Repository

A **collection-like interface** for aggregate persistence. The domain defines what it needs; infrastructure provides it.

```
// In domain layer — interface only
interface OrderRepository {
  findById(id: OrderId): Order | null
  findActiveOrders(): Order[]
  save(order: Order): void
}

// In infrastructure layer — implementation
class PostgresOrderRepository implements OrderRepository { ... }
```

**Rules:**
- Interface lives in domain layer (`domain/{context}/interfaces/`)
- Implementation lives in infrastructure layer
- One repository per aggregate root — never for non-root entities
- Use domain language in method names (`findByEmail`, `findActiveOrders`), not database operations
- Never contain business logic
- Never return partial aggregates — use read models or query services for that

#### Domain Service

A **stateless operation** involving multiple aggregates or concepts that doesn't belong to any single one.

```
// Pricing involves Product, Customer, and Promotion — belongs to none
PricingService.calculatePrice(product, customer, promotions)
```

**Use when:** operation spans multiple entities, placing it on one entity feels forced, or it represents a significant domain concept.

**Not when:** logic naturally belongs on an entity → put it there. Purely technical (email, logging) → infrastructure. Orchestration → application service.

**Domain vs Application Service:**
- **Domain Service**: business rules, lives in domain layer
- **Application Service**: use case orchestration, lives in application layer, delegates to domain

#### Domain Event

An **immutable record** of something that happened. Past tense. Immutable after creation.

- `OrderPlaced` — not `OrderCreated`, not `OrderEvent`
- `PaymentFailed` — not `PaymentError`
- `UserRegistered` — not `NewUser`

**Payload:** include aggregate ID, relevant data consumers need, timestamp. Avoid coupling consumers to the producer's internal structure — use primitives and value objects.

**Rules:**
- Aggregates raise events inside domain methods (not from application layer code)
- Application services publish raised events after persisting
- Consumers must be idempotent — they may receive events more than once

#### Factory

Encapsulates **complex object creation** requiring significant logic or coordination.

**Use when:**
- Complex validation or multi-step setup during construction
- Multiple creation paths for the same type
- Construction logic >5 lines or involves multiple entities

**Placement:**
- `domain/{context}/factories/` for domain-level construction
- Aggregate root static methods for self-creation or internal entity creation
- Infrastructure layer for reconstituting aggregates from persistence

**Design:** produce a valid object or fail — never a half-initialized object. Never a factory that can return null silently.

#### Specification

A **composable business rule** encapsulating query logic or domain constraints.

**Use when:** reusable business rules across aggregates, complex query criteria that need to live in the domain.

**Placement:** `domain/{context}/specifications/`

**Design:** composable with AND/OR/NOT. Expresses domain concepts, not technical queries.

#### Validation

Domain validators enforce business rules before aggregate construction or mutation.

**Placement:** `domain/{context}/validation/` — injected into aggregates via constructor for testability.

**Design:** validate business rules, not structural format. Throw domain exceptions on failure.

---

## Part 2 — Layer Architecture

Four layers. Dependencies point **inward only**.

```
┌──────────────────────────────────────────┐
│          Interface / Presentation        │
│  REST controllers, routes, validators,   │
│ CLI commands, error mapping, auth checks │
├──────────────────────────────────────────┤
│             Infrastructure               │
│  repository implementations, ORM,        │
│  external API clients, event publishers, │
│  caching, auth mechanisms, config        │
├──────────────────────────────────────────┤
│              Application                 │
│  use case handlers, DTOs, mappers,       │
│  port interfaces, input validation,      │
│  transaction boundaries                  │
├──────────────────────────────────────────┤
│                Domain                    │
│  aggregates, entities, value objects,    │
│  domain services, domain events,         │
│  repository interfaces, factories,       │
│  specifications, validation              │
└──────────────────────────────────────────┘
```

### Domain Layer (Innermost)

**Zero external dependencies.** No framework, database, HTTP library, ORM annotation.

**Belongs here:** aggregates, entities, value objects, factories, repository interfaces, domain services, specifications, validators, domain events.

**Does not belong here:** database queries, ORM annotations, HTTP objects, framework base classes, transaction management, file/network I/O.

**Dependency direction:** depends on nothing. Defines interfaces for outer layers to implement.

**No domain versioning.** Business rules should be version-agnostic. Multiple API versions use the same domain with different DTOs. Versioning occurs at application/interface boundaries.

**Directory:**
```
domain/
└── {bounded-context}/
    ├── aggregates/       # Aggregate roots
    ├── entities/         # Domain entities
    ├── valueObjects/     # Value objects
    ├── factories/        # Domain factories
    ├── interfaces/       # Repository and service interfaces
    ├── services/         # Domain services
    ├── specifications/   # Business rule query objects
    ├── validation/       # Domain validators
    └── events/           # Domain events
```

### Application Layer

**Orchestrates domain objects to fulfill use cases. No business logic** — delegates all decisions to the domain.

**Belongs here:** use case handlers (one per use case), DTOs, mappers (domain ↔ DTOs), port interfaces for external services, input validation (structural, not business rules), transaction boundary declarations.

**Does not belong here:** business rules, database queries, HTTP concepts (status codes, headers), complex transformations that represent domain logic.

**Dependency direction:** depends on Domain only. Defines ports (interfaces) for Infrastructure to implement.

**Version isolation:** each API version owns its complete contract independently.

**Directory:**
```
application/
├── interfaces/              # Version-independent infrastructure ports only
│   ├── EmailSenderInterface.ts
│   └── FileStorageInterface.ts
└── api/
    └── {version}/           # v1, v2, v3 — fully isolated
        ├── dtos/            # DTOs, filters, request/response for this version
        ├── mappers/         # Domain models ↔ version-specific DTOs
        ├── useCases/        # Use case orchestrators
        └── services/        # Application services (optional)
```

**Share logic, not contracts:**
- ✅ Utilities, pagination helpers → `application/shared/`
- ✅ Business rules → `domain/`
- ❌ DTOs and API contracts — duplicate per version (DRY creates hidden coupling here)

### Infrastructure Layer

**Implements ports defined by inner layers.** All external system integrations.

**Belongs here:** repository implementations, external service adapters (API clients, message brokers), ORM configuration, event publishers, caching, authentication mechanisms, environment configuration.

**Does not belong here:** business rules, use case orchestration, controller/presentation logic.

**Dependency direction:** depends on Application and Domain. Implements their interfaces.

**Directory:**
```
infrastructure/
└── repositories/
    └── {entity-name}/
        └── {version}/
            ├── entities/    # External API response/request interfaces
            ├── mappers/     # Entity ↔ Domain transformations
            └── {Entity}Repository.ts
```

### Interface / Presentation Layer (Outermost)

**Entry point translating external input into application commands/queries.**

**Belongs here:** controllers (receive input → call use case → return output), route definitions, input format validation, response DTOs, error mapping (domain exceptions → protocol responses), auth checks (is the user allowed to attempt this?).

**Does not belong here:** business logic, database access, domain object construction, complex transformations.

**Dependency direction:** depends on Application only. Never Infrastructure directly (except DI at the composition root).

**Directory:**
```
interface/
└── api/
    └── {version}/
        ├── controllers/
        ├── routes/
        ├── middleware/
        ├── validators/
        └── presenters/
```

### Composition Root

The **single place** where all layers are wired together at application startup. This is the only place that knows about all concrete implementations.

- Infrastructure implementations are bound to domain/application interfaces here
- No other code should create concrete infrastructure objects directly
- Typically the framework's DI container configuration or the application bootstrap file

---

## Part 3 — Critical Principles

### Independent Definitions Principle

**Each layer owns its own type definitions.** Never re-export or extend domain types as DTOs.

```typescript
// ❌ Re-export domain as DTO — hard coupling
export type OrderDTO = Order;

// ❌ Extend domain types — spreads coupling
export interface OrderResponse extends Order {}

// ✅ Independent definition — complete isolation
export interface OrderDTO {
  id: string;
  customerName: string;
  totalAmount: number;
  status: string;
}
```

**Why:** different layers have different concerns (behavior vs transport vs storage), need to evolve independently, and require explicit control over what crosses each boundary.

**Cost:** ~50-100 lines of mapping per entity.  
**Benefit:** independent evolution, safe refactoring, controlled exposure, type safety. This is not overhead — it is the explicit cost of architectural integrity.

### Explicit Mapping

Map explicitly at every boundary. Three boundaries require three mapping layers:

**Infrastructure ↔ Domain:**
- Translates external system representations ↔ domain aggregates
- Location: `infrastructure/repositories/{entity}/{version}/mappers/`
- Handles naming translation (snake_case → camelCase), type transformation (primitives → value objects)
- Delegates to Factory for complex aggregate creation

**Application ↔ Domain:**
- Transforms use case inputs/outputs ↔ domain models
- Location: `application/api/{version}/mappers/`
- DTOs are version-specific — v1 mapper and v2 mapper are separate classes
- Respects include directives (optional related data)
- Complex creation → delegate to Factory

**Interface ↔ Application:**
- Protocol-specific formats ↔ application DTOs
- Location: `interface/{protocol}/{version}/presenters/`
- Only needed when DTO shape doesn't match response, or for multi-protocol support

**Mapper vs Factory decision:**
- Simple (1-3 lines, format validation only) → mapper handles it directly
- Complex (>5 lines, business rules, multiple entities/value objects, conditional logic) → delegate to Factory

**Mapper rules:**
- Stateless — pure functions or static methods
- Belongs to the outer layer (infrastructure/application/interface)
- No business logic — transform, don't decide
- Handle nullability explicitly
- Never call repositories or external services

### Encapsulation Rules

Domain models are **rich, not anemic.** An anemic domain model (plain data bag with getters/setters and no behavior) is an anti-pattern.

```
// ❌ Anemic — data bag, logic in application layer
order.status = "placed"
order.updatedAt = now()
orderService.validatePlacement(order)

// ✅ Rich — behavior on the model, invariants enforced
order.place()  // validates, sets status, raises OrderPlaced event
```

Rules enforced in every domain class:
- Private constructor — no `new Aggregate()` from outside
- Static `create()` enforces validation at birth — valid or exception, never half-initialized
- No raw setters on aggregates — state changes happen through intention-revealing methods
- No `toJSON()`, `toDTO()`, `toPlainObject()` on domain objects

---

## Part 4 — API Versioning

Version at **application and interface boundaries** — never in the domain.

- `application/api/v1/` and `application/api/v2/` are fully isolated — separate DTOs, mappers, use cases
- `interface/api/v1/` and `interface/api/v2/` are fully isolated — separate controllers, validators
- Domain layer has no v1/v2 concept — business rules are version-agnostic
- Infrastructure may be versioned independently when external systems change

**Never** create a shared DTO used by both v1 and v2 routes — that hidden coupling prevents independent evolution.

---

## Part 5 — Common Violations (Red Flags)

These patterns always indicate an architectural problem:

| Violation | What It Looks Like | Why It's Wrong |
|---|---|---|
| Anemic domain model | Domain classes with only getters/setters, all logic in services | Business rules scattered, model can't enforce invariants |
| Domain model leaking to API | Controller returns domain objects directly | Internal structure exposed, domain locked to API contract |
| Shared types across boundaries | `export type UserDTO = User` | Layer coupling — one change breaks both |
| Domain importing application/infra | Domain class imports a service or ORM type | Domain becomes framework-dependent, untestable in isolation |
| Application importing infrastructure | Use case calls a concrete DB class | Business logic coupled to technology choice |
| Business logic in mapper | Mapper contains if/else domain decisions | Logic hidden in transformation code, not in domain |
| Serialization on domain objects | `order.toJSON()` or `order.toDTO()` | Domain responsible for own representation, loses independence |
| Missing mapper at boundary | Domain object passed directly to response | No control over what is exposed |
| Cross-version DTO sharing | v1 and v2 import same DTO | Versions no longer independent |
| Business logic in application service | Application handler contains domain if/else | Domain model is anemic, business rules not co-located with data |

---

## Part 6 — Compliance Verification

Before finishing implementation, verify each item:

### Import tree check
- [ ] Domain layer imports nothing from application or infrastructure
- [ ] Application layer imports nothing from infrastructure (uses interfaces only)
- [ ] Infrastructure implements interfaces — it does not define them
- [ ] Interface/Presentation layer imports from Application only (not Infrastructure directly)

### Domain model check
- [ ] All aggregate/entity changes go through aggregate root methods
- [ ] Value objects are immutable — no mutation after construction
- [ ] Domain events are raised inside aggregate methods, not from application services
- [ ] No `toJSON()`, `toDTO()`, `toPlainObject()` on domain classes
- [ ] Private constructors and static `create()` on aggregates

### Mapping check
- [ ] Independent DTO definitions exist at each boundary (no type re-exports)
- [ ] Explicit mappers present at Infrastructure↔Domain boundary
- [ ] Explicit mappers present at Application↔Domain boundary
- [ ] Complex creation (>5 lines, multiple entities) delegates to Factory, not mapper

### API versioning check
- [ ] v1 and v2 DTOs are separate definitions, no shared API contract types
- [ ] Domain layer has no v1/v2 concept

### Ubiquitous language check
- [ ] All class, method, and variable names use exact terms from the spec and domain
- [ ] No paraphrasing of domain concepts in code names
