---
name: cqs-cqrs
description: Command Query Separation (CQS) and Command Query Responsibility Segregation (CQRS). Defines the difference between commands and queries, how to structure handlers, and when to escalate from basic CQS to full CQRS with separate read/write stores.
---

# Command Query Separation and CQRS

---

## Command Query Separation (CQS)

Every operation is either a **command** or a **query**. Never both.

| | Command | Query |
|---|---|---|
| Purpose | Changes state | Reads state |
| Side effects | Yes — persists changes, publishes events | None |
| Return value | Nothing, or acknowledgment only (e.g., created ID) | Data |
| Example | `PlaceOrder`, `RegisterUser`, `CancelSubscription` | `GetOrderSummary`, `ListActiveUsers`, `FindProductById` |

This separation is not just an architectural preference — it prevents a class of bugs where a read operation unexpectedly modifies state, and it allows reads and writes to be optimized independently.

---

## Commands

A command represents an intent to change the system. It is named in the imperative: `PlaceOrder`, not `OrderPlacement` or `OrderWasPlaced`.

### Command Handler Structure

```
1. Validate input (fail fast on invalid data before touching the domain)
2. Load the aggregate from its repository
3. Call the aggregate method (business logic executes, domain events are raised)
4. Persist the aggregate
5. Dispatch the raised domain events
6. Return nothing (or the ID of a newly created resource)
```

```
class PlaceOrderHandler {
  handle(command: PlaceOrderCommand): OrderId {
    // 1. Validate
    validate(command)

    // 2. Load
    const customer = customerRepository.findById(command.customerId)
    const products = productRepository.findByIds(command.productIds)

    // 3. Execute domain logic
    const order = Order.place(customer, products)  // raises OrderPlaced event

    // 4. Persist
    orderRepository.save(order)

    // 5. Dispatch events
    eventDispatcher.dispatch(order.pullEvents())

    // 6. Return
    return order.id
  }
}
```

### Rules for Commands

- A command handler that returns rich domain data is a CQS violation — use a follow-up query
- Commands must be idempotent where possible — retrying a command should not create duplicate state
- Validate before touching the domain — do not let an invalid command partially execute

---

## Queries

A query reads and returns data. It has zero side effects.

### Query Handler Structure

```
1. Read from the read store (does not need to be the same store as the write side)
2. Project to a DTO (never return domain objects from queries)
3. Return the DTO
```

```
class GetOrderSummaryHandler {
  handle(query: GetOrderSummaryQuery): OrderSummaryDto {
    // Read directly from the read store — no aggregate loading needed
    return orderReadStore.findSummaryById(query.orderId)
  }
}
```

### Rules for Queries

- Never modify state — no saves, no event publishing, no side effects
- Return DTOs, not domain objects — DTOs are stable API contracts; domain objects are internal
- Can be served from a read-optimized store (a denormalized table, a cache, a search index)
- Can be cached — since they have no side effects, the same query with the same input always returns the same result (until the write side changes something)

---

## Directory Structure

Organize the application layer to make the separation visible:

```
src/
  application/
    commands/
      place-order/
        place-order.command.ts
        place-order.handler.ts
      register-user/
        register-user.command.ts
        register-user.handler.ts
    queries/
      get-order-summary/
        get-order-summary.query.ts
        get-order-summary.handler.ts
        order-summary.dto.ts
```

---

## Escalating to CQRS

Basic CQS (separate command and query handlers sharing the same store) is the starting point. Full CQRS — where the write side and read side use separate stores — is an optimization for specific scaling problems.

**Start with basic CQS.** Only escalate to full CQRS when you have a concrete problem:

| Problem | CQRS solution |
|---|---|
| Read queries are slow because they join normalized write tables | Maintain a denormalized read model updated by domain events |
| Write and read load require different scaling | Scale write store and read store independently |
| Need to serve multiple different read representations of the same data | Maintain multiple read models, each optimized for one use case |
| Need to replay history and rebuild state | Event sourcing: store events as the write side, project read models from them |

### CQRS with Separate Read Model

When read performance is the driver, maintain a read model updated asynchronously by domain events:

```
Write side:                          Read side:
  PlaceOrder command                   OrderPlaced event
       ↓                                    ↓
  Order aggregate                    OrderSummaryProjection
  (normalized in DB)                 (denormalized in read DB/cache)
                                           ↓
                                     GetOrderSummary query
                                     (fast, no joins)
```

The read model is eventually consistent — it updates after the event is processed. This is acceptable for most use cases. If strong consistency is required, keep both models in the same transaction (same store, same transaction — but this limits scaling).

---

## Common Violations to Watch For

- **Command that returns domain data** — either return only the ID, or follow up with a query
- **Query with a counter increment** ("get article and increment view count") — split into a query and a separate command
- **Application service that does both** — a service that reads, modifies, saves, and returns data is doing both; split it
- **Event handler that queries** — an event handler modifying state (command) that also returns data for the caller to use — the caller should issue a follow-up query
