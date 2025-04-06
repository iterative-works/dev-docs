---
status: tested
last updated: 2025-04-06
version: "1.0"
---
> [!tested] Tested Document
> This document has been validated in real projects.
# Architecture Component Classification and Organization

## Overview

This document outlines our approach to organizing code, classifying components, and structuring the application. It provides a framework for developers to consciously categorize each class or module they create, ensuring consistency across our codebase.

## Core Classification Categories

When creating a new class or component, explicitly categorize it according to the following scheme:

### Domain Layer

1. **Entity** - Objects with identity that persists across state changes
   - Has lifecycle and identity (usually via ID)
   - Immutable with versions in our FP system
   - Example: `Account`, `Transaction`, `User`

2. **Value Object** - Immutable objects defined by their attributes
   - No identity beyond their attribute values
   - Always immutable
   - Example: `Money`, `DateRange`, `Address`
   
3. **Aggregate** - Cluster of domain objects treated as a unit
   - Has a root entity that controls access
   - Maintains invariants across multiple objects
   - Example: `Order` (containing `OrderLines`, `ShippingInfo`)

4. **Domain Service** - Pure business operations that don't belong to entities
   - Stateless
   - Operates on multiple entities/value objects
   - Example: `TransactionMatcher`, `FeeCalculator`

5. **Domain Event** - Representation of something that happened in the domain
   - Immutable record of a system occurrence
   - Named in past tense
   - Example: `TransactionImported`, `AccountCreated`

### Core Abstract Components

6. **Repository Interface** - Domain-defined data access abstractions
   - Defined in domain layer
   - Expresses needed capabilities without implementation
   - Example: `TransactionRepository`, `UserRepository`

7. **Port** - Domain-defined external system interface
   - Used for outbound communication defined by the domain
   - Example: `EmailSender`, `PaymentGateway`

8. **Effect Definition** - Domain capability requiring side effects
   - Abstracts required effects in functional core
   - Example: `BankingEffect`, `NotificationEffect`

### Application Layer

9. **Application Service** - Orchestrates domain objects to fulfill use cases
   - Composes domain objects to implement features
   - Handles transactions, security, etc.
   - Example: `TransactionImportService`, `UserRegistrationService`

10. **Command/Query** - Input data structures for application services
    - DTOs used for service operations
    - Example: `ImportTransactionsCommand`, `GetAccountBalanceQuery`

11. **Result** - Output data structure from application service
    - DTOs returned from service operations 
    - Example: `ImportSummaryResult`, `AccountInfoResult`
    
12. **Validator** - Ensures input conforms to domain rules
    - Validates commands/queries before processing
    - Example: `ImportCommandValidator`, `UserInputValidator`

### Infrastructure Layer

13. **Repository Implementation** - Concrete implementation of repository interfaces
    - Implements data access defined by domain interfaces
    - Example: `PostgreSQLTransactionRepository`, `MongoUserRepository`

14. **Adapter** - Implementation of ports or connection to external systems
    - Implements domain-defined ports or connects to external systems
    - Example: `SMTPEmailSender`, `StripePaymentGateway`

15. **DTO** - Data Transfer Objects for external systems
    - Maps between domain objects and external formats
    - Example: `UserDTO`, `TransactionDTO`

16. **Infrastructure Service** - Technical services without domain knowledge
    - Handles cross-cutting concerns
    - Example: `TransactionLogger`, `MetricsCollector`

### UI/Presentation Layer

17. **Web Module** - Groups UI components for a bounded context
    - Orchestrates routes, views, and services for a feature set
    - Example: `TransactionImportModule`, `UserManagementModule`

18. **Presenter/Service** - Prepares data for views
    - Translates between domain and view models
    - Handles UI-specific business logic
    - Example: `TransactionService`, `UserProfileService`

19. **View Model** - Data structure prepared for presentation
    - Adapts domain objects for view-specific needs
    - Example: `TransactionViewModel`, `UserProfileViewModel`

20. **View** - Renders UI representation
    - Pure function that renders HTML/UI
    - No business logic
    - Example: `TransactionListView`, `UserProfileView`

21. **Routes** - HTTP endpoints
    - Maps HTTP methods/paths to services and views
    - Example: Transaction routes, User management routes

### Cross-cutting Concerns

22. **Configuration** - Application setup and parameters
    - Loads and provides configuration values
    - Example: `ApplicationConfig`, `DatabaseConfig`

23. **Module** - Wires components together (DI)
    - Handles dependency resolution
    - Example: `RepositoryModule`, `ServiceModule` 

## Code Organization: Bounded Contexts with Vertical Slices

We organize our code primarily by bounded context (domain area), with each bounded context containing its complete vertical slice of functionality. This approach keeps related functionality together while maintaining clear boundaries.

```
src/
  └── main/
      └── scala/
          └── works/iterative/
              ├── accounts/       # Bounded Context: Accounts
              │   ├── domain/     # Domain model for accounts
              │   │   ├── Account.scala
              │   │   ├── AccountRepository.scala (interface)
              │   │   └── ...
              │   ├── application/  # Application services
              │   │   ├── AccountService.scala
              │   │   └── ...
              │   ├── infrastructure/  # Implementation details
              │   │   ├── PostgreSQLAccountRepository.scala
              │   │   └── ...
              │   └── web/  # UI components
              │       ├── AccountModule.scala
              │       ├── AccountViewModel.scala
              │       ├── views/
              │       │   ├── AccountListView.scala
              │       │   └── ...
              │       └── ...
              │
              └── transactions/   # Bounded Context: Transactions
                  ├── domain/     # Domain model for transactions
                  ├── application/  # Application services
                  ├── infrastructure/  # Implementation details
                  └── web/  # UI components
```

This structure provides several key benefits:
1. **Cohesion**: Related functionality stays together
2. **Boundaries**: Clear separation between bounded contexts
3. **Navigation**: Easy to find all components related to a feature
4. **Ownership**: Teams can own entire bounded contexts
5. **Evolution**: Bounded contexts can evolve independently

## Functional MVP Pattern for Web Modules

For our web modules, we use a pattern we call "Functional MVP" (Model-View-Presenter), adapted to our functional programming approach with ZIO. This pattern maintains the separation of concerns from traditional MVP while embracing functional programming principles.

### Key Components

1. **Module Class** - Composes presenters, views, and routes
   ```scala
   class TransactionImportModule(
       appShell: ScalatagsAppShell,
       transactionListView: TransactionListView
   ) extends ZIOWebModule[Dependencies]
   ```

2. **View Model** - Adapts domain objects for the view
   ```scala
   case class TransactionViewModel(
       transaction: Transaction,
       state: Option[TransactionProcessingState]
   )
   ```

3. **Presenter/Service** - Prepares data for views
   ```scala
   class TransactionService:
     def getTransactionsWithState: ZIO[Dependencies, Throwable, 
         (Seq[TransactionViewModel], Option[String])] = ...
   ```

4. **View** - Pure rendering function
   ```scala
   class TransactionListView:
     def render(
       transactions: Seq[TransactionViewModel],
       importStatus: Option[String]
     ): Frag = ...
   ```

5. **Routes** - HTTP endpoints
   ```scala
   def routes: HttpRoutes[WebTask] = 
     HttpRoutes.of[WebTask] {
       case GET -> Root / "transactions" =>
         for
           (transactions, status) <- service.getTransactionsWithState
           response <- Ok(appShell.wrap("Transactions", 
               transactionListView.render(transactions, status)))
         yield response
     }
   ```

### For Simple Modules

For simpler features, all components can be in a single file with nested objects:

```scala
class SimpleModule extends ZIOWebModule[Dependencies]:
  case class ViewModel(...)
  
  object service:
    def getData: ZIO[Dependencies, Throwable, ViewModel] = ...
  
  object view:
    def render(data: ViewModel): Frag = ...
  
  def routes: HttpRoutes[WebTask] = ...
```

### For Complex Modules

For larger features, split the module into multiple files within the same package:

```
transactions/web/
├── TransactionImportModule.scala  # Main module + routes
├── TransactionViewModel.scala     # View models
├── TransactionService.scala       # Business logic + data access
└── views/
    ├── TransactionListView.scala  # View components
    └── ... other views ...
```

## Best Practices

### Component Creation Checklist

When creating a new class, ask yourself:

1. **Layer Identification**: Which architectural layer does this belong to?
2. **Component Type**: Which specific category from the classification list does this fall into?
3. **Responsibility Check**: Does the component have a single, well-defined responsibility?
4. **Interface Needs**: Should this be an interface in the domain layer with implementation elsewhere?
5. **Purity Assessment**: Is this component pure or does it involve side effects?

### Documentation

Add categorization in class/object documentation comments:

```scala
/**
 * Service that matches imported transactions with existing ones.
 * 
 * Category: Domain Service
 * Layer: Domain
 */
trait TransactionMatcher {
  // ...
}
```

### Boundary Enforcement

To enforce architectural boundaries:

1. **Module Structure**: Use SBT module boundaries for high-level separation
2. **Package Discipline**: Use packages to enforce boundaries within modules
3. **ArchUnit Tests**: Write tests that verify architectural rules
4. **Code Reviews**: Specifically check for boundary violations in reviews

## Conclusion

This classification and organization approach gives us a clear framework for building our application. By explicitly categorizing components and organizing code by bounded context, we ensure that related functionality stays together while maintaining clear separation of concerns. The Functional MVP pattern for our web modules provides a consistent approach to building user interfaces that aligns with our functional programming principles.

By consciously placing each class or component into the right category, we build a more maintainable, testable, and understandable codebase.

## Related Documentation

For more information on specific aspects of our architecture:
- [[domain_events_approach]]: Our pragmatic approach to implementing domain events
- [[sbt_build_bounded_contexts]]: How to structure SBT build for our architecture