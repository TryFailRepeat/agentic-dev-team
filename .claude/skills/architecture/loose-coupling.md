---
name: loose-coupling
description: Patterns for designing loosely coupled systems. Covers domain events for intra-context communication, integration events and event buses for inter-context communication, webhooks for external notifications, and anti-corruption layers for unavoidable synchronous calls.
---

# Loose Coupling

Loose coupling means components can change independently without breaking each other. The goal is not zero coupling — that's impossible. The goal is to minimize the blast radius of any change and to allow components to evolve at different rates.

---

## Within a Bounded Context: Domain Events

Aggregates must not call each other's methods directly. When one aggregate needs to react to a change in another, it does so via a domain event.

**Direct coupling (wrong):**
```
// OrderService calling InventoryAggregate directly
inventoryService.reserveItems(order.items)   // OrderService now depends on InventoryService
```

**Event-driven (correct):**
```
// Order aggregate raises an event
order.place()  // internally raises OrderPlaced { orderId, items }

// Application layer dispatches the event
eventDispatcher.dispatch(order.pullEvents())

// InventoryHandler reacts independently
class InventoryHandler {
  handle(event: OrderPlaced) {
    inventory.reserve(event.items)
  }
}
```

### Domain Event Rules

- Raised **inside** the aggregate method when state changes — not from the application layer
- Immutable — never modified after creation
- Past tense name: `OrderPlaced`, not `PlaceOrder`
- Contain only the data needed by consumers — do not include the full aggregate state
- The aggregate accumulates events; the application layer pulls and dispatches them after the aggregate is persisted

### Event Dispatcher Interface

The domain raises events via an interface. The domain never knows which technology dispatches them.

```
// In application layer (interface)
interface EventDispatcher {
  dispatch(events: DomainEvent[]): void
}

// In infrastructure (implementations, wired via DI)
class RabbitMqDispatcher implements EventDispatcher { ... }
class InMemoryDispatcher implements EventDispatcher { ... }  // for tests
```

---

## Between Bounded Contexts: Integration Events

Domain events are internal to a bounded context. When a change in one context needs to reach another context, use **integration events** — a separate concept, defined at the context boundary.

Integration events:
- Are versioned (`OrderPlacedV2`) — contexts may evolve at different rates
- Contain only the data the consuming context needs — do not leak the full domain model
- Are published to an event bus (RabbitMQ, Kafka, Redis Streams, etc.)
- Are consumed asynchronously by the receiving context

**Separation matters:**
```
// Domain event (internal, never published externally)
OrderPlaced { orderId, customerId, items: OrderItem[] }

// Integration event (crosses context boundary, on the bus)
OrderPlacedV1 { orderId, customerId, totalAmount, currency }
```

The producing context translates its domain event into an integration event — the consumer never sees internal domain types.

### Event Bus Selection

| Tool | Best for |
|---|---|
| RabbitMQ | Reliable delivery, complex routing, acknowledgments |
| Kafka | High throughput, event log, replay capability |
| Redis Streams | Low-latency, simpler setup, small teams |
| In-process event bus | Single-service, tests, simple scenarios |

When designing: choose based on delivery guarantees and operational complexity, not familiarity.

---

## External Notifications: Webhooks

When an external system needs to be notified of events, prefer webhooks over polling.

**Webhook design rules:**
- Define the payload schema explicitly (field names, types, versions)
- Include an event type field so receivers can route by event
- Sign the payload with an HMAC secret so receivers can verify authenticity
- Return `200` immediately on receipt — process asynchronously if needed
- Retry policy: document how many retries, with what backoff, and for how long

**Webhook receiver design:**
The receiver is an infrastructure adapter. It:
1. Validates the signature
2. Parses the payload
3. Translates it into a domain command
4. Hands the command to the application layer

No domain logic in the webhook receiver.

```
// Infrastructure: webhook adapter
class StripeWebhookController {
  handle(request) {
    verifySignature(request)
    const event = parseStripeEvent(request.body)

    if (event.type === 'payment_intent.succeeded') {
      commandBus.dispatch(new ConfirmPayment({ paymentId: event.data.id }))
    }
  }
}
```

---

## Unavoidable Synchronous Calls: Anti-Corruption Layer

Sometimes synchronous calls between components are unavoidable — a third-party API, a legacy service, or a real-time constraint. In these cases, use an anti-corruption layer (ACL) to prevent the external model from leaking into your domain.

The ACL is an infrastructure adapter that:
- Translates your domain's request into the external service's format
- Calls the external service
- Translates the response back into domain types
- The domain never sees the external types

```
// Domain layer: defines what it needs
interface PaymentGateway {
  charge(amount: Money, card: CardToken): PaymentResult
}

// Infrastructure: the ACL that talks to Stripe
class StripePaymentGateway implements PaymentGateway {
  charge(amount: Money, card: CardToken): PaymentResult {
    const stripeResult = stripe.paymentIntents.create({
      amount: amount.inCents(),
      currency: amount.currency,
      payment_method: card.token
    })
    return new PaymentResult(stripeResult.id, stripeResult.status)
  }
}
```

---

## When to Use Each Mechanism

| Scenario | Mechanism |
|---|---|
| Aggregate A reacts to change in Aggregate B (same context) | Domain event |
| Context A needs to notify Context B | Integration event on event bus |
| External system needs to notify your service | Webhook receiver (infrastructure adapter) |
| Your service needs to notify an external system | Webhook call or integration event publisher |
| Real-time, synchronous cross-context call unavoidable | Anti-corruption layer over a defined interface |
| Polling external state | Avoid — prefer webhooks or events |
