---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
---
> [!robot] AI-Generated Content
> This is AI-generated content pending human review. Information may be inaccurate or misaligned with actual processes.

# Implementation Guide: Repository Interface

## Purpose and Responsibilities

A Repository Interface is an abstract boundary between the domain and data access that:

- Defines persistence operations for domain entities
- Expresses data access needs in domain terms
- Follows the Repository pattern from DDD
- Enables persistence ignorance in the domain model
- Facilitates testing by allowing substitution

Repository Interfaces should:
- Express capabilities required by the domain
- Use domain language and types in their API
- Define a complete set of operations needed by domain clients
- Be defined within the domain layer
- Be implementation-agnostic

Repository Interfaces should NOT:
- Include implementation details like SQL, connection handling, etc.
- Define technology-specific methods
- Dictate how the implementation will work
- Include business logic (which belongs in domain entities or services)

## Structure and Patterns

### Basic Structure

```scala
// Basic repository interface
trait UserRepository:
  def findById(id: UserId): ZIO[Any, RepositoryError, Option[User]]
  def findByEmail(email: EmailAddress): ZIO[Any, RepositoryError, Option[User]]
  def findAll: ZIO[Any, RepositoryError, Seq[User]]
  def save(user: User): ZIO[Any, RepositoryError, User]
  def delete(id: UserId): ZIO[Any, RepositoryError, Unit]
```

### Key Structural Elements

1. **Interface Declaration**:
   - Define as a trait
   - Name as `{Entity}Repository`
   - Place in the domain layer, same package as the entity

2. **Effect Type**:
   - Use ZIO for expressing effects
   - Include appropriate error type (usually a sealed trait)
   - Make environments explicit (typically `Any` for interfaces)

3. **Method Signatures**:
   - Use domain types for parameters and returns
   - Name methods using domain language
   - Express intent rather than implementation

4. **Standard Methods**:
   - Define basic CRUD operations (find, save, delete)
   - Include domain-specific query methods
   - Consider pagination for collection returns

5. **Error Handling**:
   - Define domain-specific error types
   - Avoid technology-specific errors in the interface

## Implementation Steps

1. **Identify the Entity**:
   - Determine which entity needs persistence
   - Understand its lifecycle and retrieval needs

2. **Define Error Type**:
   - Create a sealed trait for repository errors
   - Define specific error cases

3. **Define Standard Methods**:
   - Include basic CRUD operations
   - Consider the minimal set of methods needed

4. **Add Domain-Specific Methods**:
   - Add query methods needed by domain clients
   - Focus on domain use cases, not technical capabilities

5. **Review Method Signatures**:
   - Ensure all methods use domain types
   - Check that effect types are consistent
   - Verify naming follows domain language

6. **Document Interface**:
   - Add clear documentation for each method
   - Explain error conditions

7. **Define in Domain Package**:
   - Place interface in the same package as the entity
   - Keep it separate from implementation

## Examples

### Simple Example: AccountRepository

```scala
// In domain package
sealed trait RepositoryError extends Throwable
case class EntityNotFound(id: String) extends RepositoryError
case class DuplicateEntity(id: String) extends RepositoryError
case class PersistenceError(cause: Throwable) extends RepositoryError

// Account Repository interface
trait AccountRepository:
  /**
   * Find an account by its ID.
   *
   * @param id The account ID to look up
   * @return The account if found, None otherwise
   */
  def findById(id: AccountId): ZIO[Any, RepositoryError, Option[Account]]

  /**
   * Find all accounts for a given customer.
   *
   * @param customerId The customer ID to find accounts for
   * @return A sequence of accounts
   */
  def findByCustomer(customerId: CustomerId): ZIO[Any, RepositoryError, Seq[Account]]

  /**
   * Find active accounts.
   *
   * @return All accounts with active status
   */
  def findActive(): ZIO[Any, RepositoryError, Seq[Account]]

  /**
   * Save an account. Creates a new one if it doesn't exist,
   * updates an existing one otherwise.
   *
   * @param account The account to save
   * @return The saved account
   */
  def save(account: Account): ZIO[Any, RepositoryError, Account]

  /**
   * Delete an account by its ID.
   *
   * @param id The account ID to delete
   */
  def delete(id: AccountId): ZIO[Any, RepositoryError, Unit]
```

### Complex Example: TransactionRepository

```scala
// Transaction Repository interface with more complex query methods
trait TransactionRepository:
  /**
   * Find a transaction by its ID.
   *
   * @param id The transaction ID to look up
   * @return The transaction if found, None otherwise
   */
  def findById(id: TransactionId): ZIO[Any, RepositoryError, Option[Transaction]]

  /**
   * Find all transactions for a given account within a date range.
   *
   * @param accountId The account ID to find transactions for
   * @param dateRange The date range to search within
   * @return A sequence of transactions
   */
  def findByAccountAndDateRange(
    accountId: AccountId,
    dateRange: DateRange
  ): ZIO[Any, RepositoryError, Seq[Transaction]]

  /**
   * Find transactions by status.
   *
   * @param status The transaction status to filter by
   * @param limit Maximum number of transactions to return
   * @param offset Starting position for pagination
   * @return A paginated sequence of transactions
   */
  def findByStatus(
    status: TransactionStatus,
    limit: Int,
    offset: Int
  ): ZIO[Any, RepositoryError, PaginatedResult[Transaction]]

  /**
   * Find transactions by category for a given account.
   *
   * @param accountId The account ID
   * @param category The transaction category
   * @return A sequence of transactions
   */
  def findByCategory(
    accountId: AccountId,
    category: TransactionCategory
  ): ZIO[Any, RepositoryError, Seq[Transaction]]

  /**
   * Find the total amount of transactions by category in a date range.
   *
   * @param accountId The account ID
   * @param categories The transaction categories to include
   * @param dateRange The date range to consider
   * @return A map of category to total amount
   */
  def sumByCategory(
    accountId: AccountId,
    categories: Set[TransactionCategory],
    dateRange: DateRange
  ): ZIO[Any, RepositoryError, Map[TransactionCategory, Money]]

  /**
   * Save a transaction. Creates a new one if it doesn't exist,
   * updates an existing one otherwise.
   *
   * @param transaction The transaction to save
   * @return The saved transaction
   */
  def save(transaction: Transaction): ZIO[Any, RepositoryError, Transaction]

  /**
   * Save multiple transactions in a batch.
   *
   * @param transactions The transactions to save
   * @return The saved transactions
   */
  def saveAll(transactions: Seq[Transaction]): ZIO[Any, RepositoryError, Seq[Transaction]]

  /**
   * Delete a transaction by its ID.
   *
   * @param id The transaction ID to delete
   */
  def delete(id: TransactionId): ZIO[Any, RepositoryError, Unit]

// Support classes
case class PaginatedResult[T](
  items: Seq[T],
  totalCount: Long,
  pageNumber: Int,
  pageSize: Int,
  hasMore: Boolean
)
```

## Integration Points

Repository Interfaces typically integrate with:

1. **Domain Entities**:
   - Repositories manage persistence for specific entities
   - They return and accept domain entity types

2. **Domain Services**:
   - Services use repositories to access and persist entities
   - Repositories provide data access capabilities to services

3. **Application Services**:
   - Application services orchestrate repositories for use cases
   - They handle transaction boundaries around repository operations

4. **Repository Implementations**:
   - Implementation classes in the infrastructure layer fulfill the interface
   - Concrete implementations connect to actual databases

5. **Tests**:
   - Mock implementations support unit testing
   - Test implementations verify behavior in integration tests

## Testing Approach

Since repository interfaces are just contracts, they are tested through their implementations:

1. **Mock Testing**:
   - Create mock implementations for unit testing domain services
   - Verify domain logic correctly uses repository methods

2. **Implementation Testing**:
   - Test concrete implementations against actual databases
   - Verify CRUD operations and query methods work correctly

3. **Integration Testing**:
   - Test repositories with domain services and application services
   - Verify end-to-end flows involving persistence

### Example Mock Implementation for Testing:

```scala
import zio.ZIO

class InMemoryUserRepository extends UserRepository:
  private var users: Map[UserId, User] = Map.empty

  override def findById(id: UserId): ZIO[Any, RepositoryError, Option[User]] =
    ZIO.succeed(users.get(id))

  override def findByEmail(email: EmailAddress): ZIO[Any, RepositoryError, Option[User]] =
    ZIO.succeed(users.values.find(_.email == email))

  override def findAll: ZIO[Any, RepositoryError, Seq[User]] =
    ZIO.succeed(users.values.toSeq)

  override def save(user: User): ZIO[Any, RepositoryError, User] =
    ZIO.succeed {
      users = users + (user.id -> user)
      user
    }

  override def delete(id: UserId): ZIO[Any, RepositoryError, Unit] =
    ZIO.succeed {
      users = users - id
    }
```

## Common Pitfalls

1. **Implementation Details**:
   - Don't include database-specific operations in the interface
   - Avoid dependencies on specific technologies

2. **Missing Domain Perspective**:
   - Don't design based on technical capabilities
   - Focus on domain use cases and language

3. **Overly Generic Methods**:
   - Don't rely on generic query mechanisms that bypass domain semantics
   - Prefer specific, named methods over general-purpose query methods

4. **Exposing Persistence Concepts**:
   - Don't expose persistence concepts like sessions or connections
   - Keep the interface focused on entity operations

5. **Synchronous APIs**:
   - Don't use blocking operations in a ZIO-based system
   - Always use effect types for potentially blocking operations

6. **Error Handling Gaps**:
   - Don't use generic exception types
   - Define specific error types for different failure scenarios

7. **Missing Pagination**:
   - Don't return unbounded collections for queries that might return large results
   - Add pagination parameters for methods that return collections

## Checklist

When implementing a repository interface, ensure:

- [ ] Interface is defined in the domain layer
- [ ] Named according to convention (`{Entity}Repository`)
- [ ] All methods use domain types for parameters and returns
- [ ] ZIO effect type used for all operations
- [ ] Clear error types defined
- [ ] Basic CRUD operations included
- [ ] Domain-specific query methods added
- [ ] Method names follow domain language
- [ ] Documentation explains purpose and error conditions
- [ ] No implementation details or technology dependencies
- [ ] Pagination for methods returning collections
- [ ] Interface meets all needs of domain clients
