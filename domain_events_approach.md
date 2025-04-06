---
status: in-review
last updated: 2025-04-06
version: "0.1"
---
> [!review] In-Review Document
> This document is under review and feedback is welcome.
# Domain Events Approach

## Pragmatic Event Implementation

While our architecture recognizes domain events as a key component, we take a pragmatic approach to their implementation:

### Core Principles

1. **Domain Events as First-Class Concepts**: We model significant state changes in our domain as explicit events
   
2. **Minimal Initial Implementation**: We start with simple event mechanisms using basic ZIO primitives
   
3. **Evolutionary Adoption**: We expand event usage and sophistication as the application grows

### Implementation Guidelines

#### 1. Event Definition

```scala
// Simple immutable event with essential data
case class TransactionImported(
  transactionId: TransactionId,
  accountId: AccountId,
  amount: Money,
  date: LocalDate,
  timestamp: Instant = Instant.now()
)
```

#### 2. Event Publication

For simple bounded contexts, use lightweight ZIO mechanisms:

```scala
// Using ZIO Hub for basic pub/sub within a bounded context
class TransactionService(eventHub: Hub[DomainEvent]):
  def importTransaction(transaction: Transaction): ZIO[Any, Error, TransactionId] =
    for
      id <- transactionRepository.create(transaction)
      _ <- eventHub.publish(TransactionImported(id, transaction.accountId, 
                   transaction.amount, transaction.date))
    yield id
```

#### 3. Event Consumption

Subscribe to events where needed:

```scala
class NotificationService(eventHub: Hub[DomainEvent]):
  val start: ZIO[Scope, Nothing, Unit] =
    eventHub.subscribe.foreach {
      case TransactionImported(txId, accountId, amount, date, _) if amount > largeAmount =>
        sendLargeTransactionAlert(txId, accountId, amount)
      case _ => ZIO.unit
    }
```

#### 4. Cross-Bounded Context Communication

For events that cross bounded context boundaries, consider:

```scala
// A service interface for publishing events across bounded contexts
trait EventPublisher:
  def publish(event: DomainEvent): UIO[Unit]

// Implementation could use direct ZIO effects initially, 
// and evolve to message queues or event streams when needed
```

### Event Storage vs. Event Sourcing

Our approach distinguishes between:

1. **Event Notification**: Using events for communication between components
2. **Event Storage**: Persisting events for audit or traceability
3. **Event Sourcing**: Using events as the primary source of truth

We generally begin with event notification only, add selective event storage where valuable for auditing or compliance, and consider full event sourcing only for aggregates where the complete history is essential to the domain.

### When to Use Events

Consider emitting domain events when:

- A significant state change occurs in an aggregate
- Other components or bounded contexts need to react to the change
- The action needs to be recorded for audit or compliance
- The operation might trigger eventual consistency updates

Not every CRUD operation needs to be an event. Focus on meaningful domain occurrences.

### Future Evolution

This approach allows us to start simply while keeping options open for more sophisticated event patterns:

- Event storage and replay capabilities
- Distributed event processing
- Integration with external event streams
- Selective event sourcing for appropriate aggregates

By making events a foundational concept but implementing them minimally, we balance immediate simplicity with future flexibility.