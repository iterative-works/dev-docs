---
status: in-review
created: 2025-05-13
last updated: 2025-04-15
version: "1.1"
---

# Implementation Guide: Domain-Driven Repository Implementation

## Purpose and Responsibilities

A Repository Implementation is a concrete infrastructure component that:

- Implements a Repository Interface defined in the domain layer
- Connects domain entities to the actual data storage mechanism
- Translates between domain objects and persistence formats
- Encapsulates all data access logic and SQL queries
- Shields the domain from database specifics and technology details

Repository Implementations should:
- Fulfill all methods defined in the corresponding Repository Interface
- Handle persistence concerns efficiently
- Translate between domain entities and database records
- Manage infrastructure-specific error handling
- Optimize data access patterns when needed

Repository Implementations should NOT:
- Define business logic (belongs in domain entities/services)
- Expose database-specific details to clients
- Use any persistence concepts in their public interfaces
- Bypass domain entities by returning raw data structures

## Repositories in Domain-Driven Design

In alignment with our Functional Core/Imperative Shell architecture and Domain-Driven Design principles, repositories serve as the boundary between the pure domain model and the infrastructure layer. They act as a collection-like interface for accessing domain objects while hiding the details of the underlying persistence mechanism.

### Domain-Defined Repository Interfaces

Repository interfaces should be defined in the domain layer and express exactly what the domain needs:

```scala
// In domain layer
trait UserRepository:
  // Find methods
  def findById(id: UserId): IO[UserNotFoundError, User]
  def findByEmail(email: EmailAddress): IO[Nothing, Option[User]]
  def findAll: IO[Nothing, Seq[User]]
  def findByFilters(filter: UserFilter): IO[Nothing, Seq[User]]

  // Create/Update methods
  def create(user: NewUser): IO[UserCreationError, UserId]
  def update(user: User): IO[UserUpdateError, Unit]
  def delete(id: UserId): IO[UserNotFoundError, Unit]
```

Note how the repository interface:
- Uses domain types (UserId, User, EmailAddress)
- Returns domain-specific errors when appropriate
- Expresses domain-relevant operations

### Error Handling in Repositories

Domain repository interfaces should express meaningful errors that are relevant to the domain. Instead of using `UIO` (which assumes nothing can go wrong), use typed errors to express domain-relevant failure modes:

```scala
sealed trait UserError
case object UserNotFoundError extends UserError
case class UserCreationError(reason: String) extends UserError
case class UserUpdateError(reason: String) extends UserError
```

This enables the domain to handle specific error cases appropriately:

```scala
def promoteUser(userId: UserId): IO[UserError, Unit] =
  for
    user <- userRepository.findById(userId)
    updatedUser = user.copy(role = Role.Admin)
    _ <- userRepository.update(updatedUser)
  yield ()
```

## CQRS Principles for Repositories

Our repository design follows these CQRS (Command Query Responsibility Segregation) principles:

1. **Commands Don't Return Domain Data**
   - Commands like `create`, `update`, and `delete` return minimal data (e.g., generated IDs) or just confirmation
   - Maintains separation between commands and queries

2. **Creation Returns Only Keys or Success/Failure**
   - `create` returns the generated key/ID or appropriate error
   - Requires explicit query to retrieve the created entity if needed

3. **Queries Don't Modify State**
   - `find` and other read methods are pure and don't modify state
   - No side effects in query operations

4. **Event Notification via Streams**
   - Consider providing update streams for reactive patterns
   - Subscribers can react to repository changes

## Implementation with Magnum Library

Our repositories are primarily implemented using the Magnum library, which provides type-safe SQL access that integrates well with our ZIO-based architecture.

### Magnum Repository Types

Magnum provides two main types of repositories:

#### 1. ImmutableRepo - Read-Only Operations

```scala
// Definition in companion object
object UserRepo:
  val repo = ImmutableRepo[UserDTO, Long]
```

**Available Methods:**
- `count(using DbCon): Long` - Count all entities
- `existsById(id: ID)(using DbCon): Boolean` - Check if entity exists
- `findAll(using DbCon): Vector[E]` - Get all entities
- `findAll(spec: Spec[E])(using DbCon): Vector[E]` - Get entities matching specification
- `findById(id: ID)(using DbCon): Option[E]` - Find entity by ID
- `findAllById(ids: Iterable[ID])(using DbCon): Vector[E]` - Find entities by IDs

#### 2. Repo - Full CRUD Operations

```scala
// Definition in companion object
object TransactionRepo:
  // For tables with no separate creator class
  val fullRepo = Repo[TransactionDTO, TransactionDTO, Null]

  // For tables with auto-generated IDs
  val userRepo = Repo[UserCreator, User, Long]
```

The `Repo` class takes three type parameters:
1. **Entity-Creator (EC)**: Type used for creating new entities (typically omits auto-generated fields)
2. **Entity (E)**: The full entity type
3. **ID**: Type of the primary key (or Null if there's no logical ID)

**Additional Methods:**
- `delete(entity: E)(using DbCon): Unit` - Delete an entity
- `deleteById(id: ID)(using DbCon): Unit` - Delete by ID
- `truncate()(using DbCon): Unit` - Delete all entities
- `deleteAll(entities: Iterable[E])(using DbCon): BatchUpdateResult` - Delete multiple entities
- `deleteAllById(ids: Iterable[ID])(using DbCon): BatchUpdateResult` - Delete by multiple IDs
- `insert(entityCreator: EC)(using DbCon): Unit` - Insert new entity
- `insertAll(entityCreators: Iterable[EC])(using DbCon): Unit` - Insert multiple entities
- `insertReturning(entityCreator: EC)(using DbCon): E` - Insert and return created entity
- `insertAllReturning(entityCreators: Iterable[EC])(using DbCon): Vector[E]` - Insert multiple and return
- `update(entity: E)(using DbCon): Unit` - Update existing entity
- `updateAll(entities: Iterable[E])(using DbCon): BatchUpdateResult` - Update multiple entities

## Structure of Repository Implementations

### Domain-Aligned Implementation with Magnum

```scala
import com.augustnagro.magnum.*
import com.augustnagro.magnum.magzio.*
import zio.*

class PostgreSQLUserRepository(xa: Transactor) extends UserRepository:
  import UserDTO.given
  import UserRepo.repo

  override def findById(id: UserId): IO[UserNotFoundError, User] =
    xa.connect:
      repo.findById(id.value).map(_.map(_.toDomain))
    .orDieWith(_ => new RuntimeException("Database error"))
    .flatMap:
      case Some(user) => ZIO.succeed(user)
      case None => ZIO.fail(UserNotFoundError(id))

  override def findByEmail(email: EmailAddress): IO[Nothing, Option[User]] =
    xa.connect:
      // Use Magnum's Spec for dynamic queries
      val spec = Spec[UserDTO].where(sql"email = ${email.value}")
      repo.findAll(spec).headOption.map(_.map(_.toDomain))
    .orDie

  override def findAll: IO[Nothing, Seq[User]] =
    xa.connect:
      repo.findAll.map(_.map(_.toDomain).toSeq)
    .orDie

  override def update(user: User): IO[UserUpdateError, Unit] =
    xa.transact:
      repo.update(UserDTO.fromDomain(user))
    .mapError(e => UserUpdateError(e.getMessage))

  override def create(user: NewUser): IO[UserCreationError, UserId] =
    xa.transact:
      val dto = UserDTO.fromNewUser(user)
      repo.insertReturning(dto).map(created => UserId(created.id))
    .mapError(e => UserCreationError(e.getMessage))

  override def delete(id: UserId): IO[UserNotFoundError, Unit] =
    findById(id).catchAll(_ => ZIO.fail(UserNotFoundError(id))) *>
    xa.transact:
      repo.deleteById(id.value)
    .mapError(_ => UserNotFoundError(id))

object UserRepository:
  val layer: ZLayer[Transactor, Nothing, UserRepository] =
    ZLayer.fromFunction { (xa: Transactor) =>
      PostgreSQLUserRepository(xa)
    }

  // Repo definition in companion object
  object UserRepo:
    val repo = Repo[UserDTO, UserDTO, Long]
```

### Data Transfer Objects (DTOs)

We use DTOs to separate our domain models from the database representation:

```scala
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
case class UserDTO(
  @Id id: Long,
  email: String,
  name: String,
  hashedPassword: String,
  role: String,
  status: String,
  createdAt: Instant,
  updatedAt: Instant
) derives DbCodec:
  // Conversion to domain model
  def toDomain: User = this.into[User]
    .withFieldComputed(_.id, dto => UserId(dto.id))
    .withFieldComputed(_.email, dto => EmailAddress.fromString(dto.email).get)
    .withFieldComputed(_.role, dto => UserRole.valueOf(dto.role))
    .withFieldComputed(_.status, dto => UserStatus.valueOf(dto.status))
    .transform

object UserDTO:
  // Conversion from domain model
  def fromDomain(user: User): UserDTO = user.into[UserDTO]
    .withFieldComputed(_.id, _.id.value)
    .withFieldComputed(_.email, _.email.value)
    .withFieldComputed(_.role, _.role.toString)
    .withFieldComputed(_.status, _.status.toString)
    .transform

  // Conversion from creation DTO
  def fromNewUser(user: NewUser): UserDTO =
    UserDTO(
      id = 0, // Will be generated by database
      email = user.email.value,
      name = user.name,
      hashedPassword = user.hashedPassword,
      role = UserRole.User.toString,
      status = UserStatus.Active.toString,
      createdAt = Instant.now,
      updatedAt = Instant.now
    )
```

## Common Repository Patterns

### Entity Creation Pattern

```scala
// Domain model
case class Account(id: AccountId, accountNumber: String, /* other fields */)

// Creation DTO for domain layer
case class NewAccount(accountNumber: String, /* other fields - no ID */)

// In repository interface (domain layer)
trait AccountRepository:
  def create(account: NewAccount): IO[AccountCreationError, AccountId]
  def findById(id: AccountId): IO[AccountNotFoundError, Account]

// In service layer
def createNewAccount(account: NewAccount): IO[AccountCreationError, AccountId] =
  for
    id <- accountRepository.create(account)
    _ <- logService.info(s"Created account with ID: $id")
  yield id

// If you need the created entity
def createAndRetrieveAccount(account: NewAccount): IO[AccountError, Account] =
  for
    id <- accountRepository.create(account)
    entity <- accountRepository.findById(id)
  yield entity
```

### Entity Update Pattern

```scala
// In repository interface (domain layer)
trait AccountRepository:
  def update(account: Account): IO[AccountUpdateError, Unit]
  def findById(id: AccountId): IO[AccountNotFoundError, Account]

// In service layer
def updateAccount(id: AccountId, updateData: AccountUpdateData): IO[AccountError, Unit] =
  for
    existingAccount <- accountRepository.findById(id)
    updatedAccount = existingAccount.update(updateData)
    _ <- accountRepository.update(updatedAccount)
  yield ()
```

### Query Pattern

```scala
// Domain query model
case class AccountFilter(status: Option[AccountStatus] = None, /* other filters */)

// In repository interface (domain layer)
trait AccountRepository:
  def findByFilters(filter: AccountFilter): IO[Nothing, Seq[Account]]

// In service layer
def findActiveAccounts(): IO[Nothing, Seq[Account]] =
  accountRepository.findByFilters(AccountFilter(status = Some(AccountStatus.Active)))

// Implementation using Magnum Spec
override def findByFilters(filter: AccountFilter): IO[Nothing, Seq[Account]] =
  xa.connect:
    // Build dynamic query with Magnum's Spec
    val spec = Spec[AccountDTO]
      .where(
          filter.status.map(status =>
              sql"status = ${status.toString}"
          ).getOrElse(sql"")
      )
      // Add other conditions as needed
      .limit(filter.limit.getOrElse(100))
      .offset(filter.offset.getOrElse(0))

    // Execute and convert to domain models
    repo.findAll(spec).map(_.map(_.toDomain).toSeq)
  .orDie
```

## Infrastructure Setup and Transactor Configuration

Our project provides standardized infrastructure components for PostgreSQL database access that should be used for repository implementations.

### PostgreSQL Infrastructure Components

We have three main infrastructure components for database access:

1. **PostgreSQLDataSource** - Manages the HikariCP connection pool for PostgreSQL
2. **PostgreSQLTransactor** - Wraps a Magnum Transactor for performing database operations
3. **PostgreSQLDatabaseSupport** - Provides combined layers and migration utilities

### Standard Repository Layer Structure

For all repository implementations, follow this layer structure pattern:

```scala
object PostgreSQLAccountRepository:
    // Repository layer taking only the PostgreSQLTransactor dependency
    val layer: ZLayer[PostgreSQLTransactor, Nothing, AccountRepository] =
        ZLayer.fromFunction { (ts: PostgreSQLTransactor) =>
            PostgreSQLAccountRepository(ts.transactor)
        }

    // Full layer including all dependencies from scratch
    val fullLayer: ZLayer[Scope, Throwable, AccountRepository] =
        PostgreSQLDataSource.managedLayer >>>
            PostgreSQLTransactor.managedLayer >>>
            layer
```

This pattern provides both a targeted layer that only depends on the transactor (useful for integration testing) and a complete layer that sets up the entire database stack.

### Database Migrations with Flyway

We use Flyway for database migrations, providing a structured way to evolve the database schema. Our project includes a `FlywayMigrationService` that wraps Flyway's functionality and integrates with our ZIO environment:

```scala
trait FlywayMigrationService:
    def migrate(): Task[MigrateResult]  // Apply pending migrations
    def clean(): Task[Unit]             // Clean/reset the database
    def validate(): Task[Unit]          // Validate migrations
    def info(): Task[Unit]              // Print migration info
```

The migration service is configured with a `FlywayConfig` that specifies migration locations and other options:

```scala
case class FlywayConfig(
    locations: List[String] = List(FlywayConfig.DefaultLocation),
    cleanDisabled: Boolean = true
)

object FlywayConfig:
    val DefaultLocation = "classpath:db/migration"
    val default: FlywayConfig = FlywayConfig()
```

### Using PostgreSQLDatabaseSupport for Migrations

For production code that requires migrations, use the `PostgreSQLDatabaseSupport` to ensure migrations run before database access:

```scala
// Example of a repository layer that requires migrations to run first
val repositoryLayerWithMigrations: ZLayer[Scope, Throwable, AccountRepository] =
    PostgreSQLDatabaseSupport.layerWithMigrations() >>>
        PostgreSQLAccountRepository.layer

// To specify additional migration locations
val repositoryLayerWithCustomMigrations: ZLayer[Scope, Throwable, AccountRepository] =
    PostgreSQLDatabaseSupport.layerWithMigrations(
        additionalLocations = List("classpath:db/specific-migrations")
    ) >>> PostgreSQLAccountRepository.layer
```

The `layerWithMigrations` method:
1. Creates a PostgreSQL data source
2. Sets up a Flyway migration service
3. Runs all migrations before providing the infrastructure
4. Returns a combined layer with both the database infrastructure and migration service

### Migration File Organization

Flyway migrations should be placed in the `src/main/resources/db/migration` directory with file names following the format:

- `V{version}__{description}.sql` - For versioned migrations (e.g., `V1__create_users_table.sql`)
- `R__{description}.sql` - For repeatable migrations (e.g., `R__views.sql`)

Versioned migrations run in order based on the version number, while repeatable migrations run whenever their content changes.

### PostgreSQLDataSource Implementation

The `PostgreSQLDataSource` class manages a connection pool using HikariCP:

```scala
class PostgreSQLDataSource(val dataSource: DataSource)

object PostgreSQLDataSource:
    // Initialize a HikariCP data source with configuration
    def initDataSource(config: PostgreSQLConfig): ZIO[Scope, Throwable, HikariDataSource] = ...

    // Layer that creates a DataSource from configuration
    val layer: ZLayer[Scope, Throwable, DataSource] = ...

    // Layer that creates a managed PostgreSQLDataSource instance
    val managedLayer: ZLayer[Scope, Throwable, PostgreSQLDataSource] =
        layer >>> ZLayer.fromFunction(PostgreSQLDataSource.apply)
```

### PostgreSQLTransactor Implementation

The `PostgreSQLTransactor` creates and manages a Magnum Transactor:

```scala
class PostgreSQLTransactor(val transactor: Transactor)

object PostgreSQLTransactor:
    // Layer that creates a Transactor from a PostgreSQLDataSource
    val layer: ZLayer[PostgreSQLDataSource & Scope, Throwable, Transactor] =
        ZLayer.service[PostgreSQLDataSource].flatMap { env =>
            Transactor.layer(env.get[PostgreSQLDataSource].dataSource)
        }

    // Layer that creates a managed PostgreSQLTransactor instance
    val managedLayer: ZLayer[PostgreSQLDataSource & Scope, Throwable, PostgreSQLTransactor] =
        layer >>> ZLayer.fromFunction(PostgreSQLTransactor.apply)
```

For repository implementations, always use the `PostgreSQLTransactor.managedLayer` to create the transactor and inject it into your repository implementation. This ensures consistent configuration and connection pooling across all repositories.

## Dynamic Queries with Magnum Spec

Use Magnum's `Spec` class for building dynamic queries with multiple optional filter conditions:

```scala
override def findByFilters(filter: TransactionFilter): IO[Nothing, Seq[Transaction]] =
  xa.connect:
    val spec = Spec[TransactionDTO]
      // Add conditions only when filter values are present
      .where(
          filter.accountId.map(id =>
              sql"account_id = ${id.value}"
          ).getOrElse(sql"")
      )
      .where(
          filter.dateFrom.map(date =>
              sql"date >= ${java.sql.Date.valueOf(date)}"
          ).getOrElse(sql"")
      )
      .where(
          filter.dateTo.map(date =>
              sql"date <= ${java.sql.Date.valueOf(date)}"
          ).getOrElse(sql"")
      )
      .where(
          filter.status.map(status =>
              sql"status = ${status.toString}"
          ).getOrElse(sql"")
      )
      // Add pagination if needed
      .limit(filter.limit.getOrElse(100))
      .offset(filter.offset.getOrElse(0))
      // Add sorting
      .orderBy(sql"date DESC, created_at DESC")

    // Execute and map results
    repo.findAll(spec).map(_.map(_.toDomain).toSeq)
  .orDie
```

## Testing Approach

We have standardized infrastructure for testing repositories using TestContainers, Flyway migrations, and ZIO Test.

### Testing Infrastructure

Our `iw-support-sqldb-testing` module provides several key components for testing repositories:

1. **PostgreSQLTestingLayers** - Provides ZLayers for test database infrastructure
2. **MigrateAspects** - Test aspects for database schema setup and teardown

### Standard Repository Testing Pattern

Use this pattern for testing repository implementations using ZIO Test:

```scala
import zio.*
import zio.test.*
import works.iterative.sqldb.testing.PostgreSQLTestingLayers.*
import works.iterative.sqldb.testing.MigrateAspects.*

object PostgreSQLAccountRepositorySpec extends ZIOSpecDefault:
    // Define layers needed for repository tests
    val repositoryLayer =
        PostgreSQLAccountRepository.layer

    def spec = {
        suite("PostgreSQLAccountRepository")(
            test("should save and retrieve an account") {
                for
                    // Get repository services
                    repo <- ZIO.service[AccountRepository]

                    // Create test data
                    newAccount = NewAccount("12345", "Test Account")

                    // Execute operations
                    id <- repo.create(newAccount)
                    retrieved <- repo.findById(id)
                yield
                    // Assert results
                    assertTrue(
                        retrieved.accountNumber == "12345",
                        retrieved.name == "Test Account"
                    )
            },

            test("should find accounts by filter") {
                for
                    repo <- ZIO.service[AccountRepository]

                    // Create test data
                    newAccount1 = NewAccount("12345", "Active Account", status = AccountStatus.Active)
                    newAccount2 = NewAccount("67890", "Inactive Account", status = AccountStatus.Inactive)

                    // Save test data
                    id1 <- repo.create(newAccount1)
                    id2 <- repo.create(newAccount2)

                    // Query with filter
                    results <- repo.findByFilters(AccountFilter(
                        status = Some(AccountStatus.Active)
                    ))
                yield
                    assertTrue(
                        results.size == 1,
                        results.head.accountNumber == "12345"
                    )
            },

            test("should handle domain-specific errors") {
                for
                    repo <- ZIO.service[AccountRepository]

                    // Try to find non-existent account
                    result <- repo.findById(AccountId(999999)).either
                yield
                    assertTrue(
                        result.isLeft,
                        result.left.toOption.exists(_.isInstanceOf[AccountNotFoundError])
                    )
            }
        // Apply aspects for all tests
        ) @@ sequential @@ migrate
    }.provideSomeShared[Scope](
        // Provide infrastructure layers
        flywayMigrationServiceLayer,
        repositoryLayer
    )
end PostgreSQLAccountRepositorySpec
```

### Key Testing Infrastructure Components

#### PostgreSQLTestingLayers

Provides ZLayers for TestContainers and database infrastructure:

```scala
object PostgreSQLTestingLayers:
    // Container for PostgreSQL test database
    val postgresContainer: ZLayer[Scope, Throwable, PostgreSQLContainer] = ...

    // DataSource connected to test container
    val dataSourceLayer: ZLayer[Scope, Throwable, DataSource] = ...

    // PostgreSQLDataSource layer
    val postgreSQLDataSourceLayer: ZLayer[Scope, Throwable, PostgreSQLDataSource] = ...

    // Transactor layer for database operations
    val transactorLayer: ZLayer[Scope, Throwable, Transactor] = ...

    // Combined layer with both DataSource and Transactor
    val postgreSQLTransactorLayer: ZLayer[Scope, Throwable, PostgreSQLDataSource & PostgreSQLTransactor] = ...

    // Test Flyway config that allows cleaning the database
    val testFlywayConfig = FlywayConfig(
        locations = FlywayConfig.DefaultLocation :: Nil,
        cleanDisabled = false
    )

    // Full layer with DataSource, Transactor and FlywayMigrationService
    val flywayMigrationServiceLayer: ZLayer[
        Scope,
        Throwable,
        PostgreSQLDataSource & PostgreSQLTransactor & FlywayMigrationService
    ] = ...
end PostgreSQLTestingLayers
```

#### MigrateAspects

Provides test aspects for database setup and teardown:

```scala
object MigrateAspects:
    // Set up fresh database schema for tests
    val setupDbSchema = ZIO.scoped {
        for
            migrationService <- ZIO.service[FlywayMigrationService]
            // Clean the database to ensure a fresh state
            _ <- migrationService.clean()
            // Run migrations to set up the schema
            result <- migrationService.migrate()
        yield ()
    }

    // Test aspect that runs migrations before tests
    val migrate = TestAspect.before(setupDbSchema)
end MigrateAspects
```

### Testing Best Practices

1. **Clean State per Test** - Apply the `migrate` aspect to ensure each test suite starts with a clean database
2. **Sequential Tests** - Use `sequential` aspect to prevent test interference when accessing the database
3. **Layer Structure** - Separate repository layers from infrastructure layers for flexible testing
4. **Realistic Data** - Test with realistic data samples that cover edge cases
5. **Query Testing** - Test complex queries with various filter combinations
6. **Error Handling** - Test domain-specific error scenarios thoroughly

This approach ensures that repository implementations are thoroughly tested against real PostgreSQL databases with proper schema migrations, while keeping the tests maintainable and reliable.

## Magnum Import Guidelines

When using Magnum with ZIO, be careful with imports to avoid ambiguity errors:

```scala
// ✅ RECOMMENDED: Specific, controlled imports
import com.augustnagro.magnum.magzio.*
import com.augustnagro.magnum.{Spec, Repo}

// ❌ AVOID: Wildcard imports that can cause conflicts
import com.augustnagro.magnum.*
import com.augustnagro.magnum.magzio.*
```

For SQL string interpolation, import only what you need:

```scala
// ✅ RECOMMENDED: Import only the sql interpolator
import com.augustnagro.magnum.sql

// In method body:
val spec = Spec[MyDTO].where(sql"my_column = $value")
```

When using `orderBy`, prefer the string version with sort order parameter over sql interpolation:

```scala
// ✅ RECOMMENDED:
.orderBy("column_name", SortOrder.Desc)

// ❌ AVOID:
.orderBy(sql"column_name DESC")
```

## Repository Method Implementation Patterns

### Method Naming Convention

Use a clear naming convention for repository methods:

- `find*` - Methods that return `Option[Entity]` or `Seq[Entity]` with no domain error
- `get*` - Methods that return `Entity` and can fail with domain error if entity doesn't exist

```scala
// Option-returning find method (no domain error)
def findById(id: Long): ZIO[Any, Nothing, Option[Entity]] =
    xa.connect {
        repo.findById(id).map(_.toDomain)
    }.orDie

// Entity-returning get method (domain error if not found)
def getById(id: Long): ZIO[Any, DomainError, Entity] =
    findById(id).flatMap {
        case Some(entity) => ZIO.succeed(entity)
        case None => ZIO.fail(EntityNotFoundError(id))
    }
```

### Query Method Pattern

Always map DTO to domain model inside the database transaction:

```scala
def findById(id: Long): ZIO[Any, Nothing, Option[Entity]] =
    xa.connect {
        // Map to domain model inside the database operation
        repo.findById(id).map(_.toDomain)
    }.orDie
```

### Collection Query Pattern

For methods returning collections, handle the mapping inside the database operation:

```scala
def getAllPublished(limit: Int = 100): ZIO[Any, Nothing, Seq[Entity]] =
    xa.connect {
        val spec = Spec[EntityDTO]
            .where(sql"published = true")
            .orderBy("created_at", SortOrder.Desc)
            .limit(limit)
        repo.findAll(spec).map(_.toDomain)
    }.orDie
```

### Creation vs. Full Entity DTOs

Use separate DTOs for creation operations and entity representation:

```scala
// DTO for database representation
@SqlName("entities")
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
case class EntityDTO(
    @Id id: Long,
    name: String,
    createdAt: Timestamp
) derives DbCodec:
    def toDomain: Entity = Entity(id, name, createdAt.toLocalDateTime)

// DTO for creating new entities
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
case class CreateEntityDTO(
    name: String,
    createdAt: Timestamp
)

object EntityDTO:
    // Convert from domain to DTO
    def fromDomain(entity: Entity): EntityDTO = 
        EntityDTO(entity.id, entity.name, Timestamp.valueOf(entity.createdAt))
        
    // Create DTO from creation data
    def fromCreate(create: CreateEntity): CreateEntityDTO =
        CreateEntityDTO(create.name, Timestamp.valueOf(LocalDateTime.now()))
```

### Validating Uniqueness Within Transactions

Keep all database operations within a single transaction for consistency:

```scala
def save(entity: CreateEntity): ZIO[Any, DomainError, Long] =
    // Validate entity according to domain rules
    ZIO.fromEither(entity.validate).flatMap(insertEntity)

private def insertEntity(entity: CreateEntity): ZIO[Any, DomainError, Long] =
    xa.transact {
        // Do all validation within the transaction
        validateUniqueName(entity.name) match
            case Left(err) => throw err
            case _         => repo.insertReturning(EntityDTO.fromCreate(entity)).id
    }.refineToOrDie[DomainError]

// Helper method that takes implicit database connection
private def validateUniqueName(name: String)(using DbCon): Either[DomainError, Unit] =
    val spec = Spec[EntityDTO].where(sql"name = $name")
    val exists = repo.findAll(spec).nonEmpty
    if exists then Left(DomainError.DuplicateNameError(name)) else Right(())
```

### Error Handling with refineToOrDie

Use `refineToOrDie` to handle expected domain errors while treating other exceptions as defects:

```scala
def update(update: UpdateEntity): ZIO[Any, DomainError, Unit] =
    xa.transact {
        val process = for
            currentDTO <- repo.findById(update.id).toRight(EntityNotFoundError(update.id))
            current = currentDTO.toDomain
            _ <- validateUpdate(current, update)
            updated = update.applyTo(current)
        yield repo.update(EntityDTO.fromDomain(updated))
        
        process match
            case Left(err) => throw err
            case _         => ()
    }.refineToOrDie[DomainError]
```

## Repository Companion Object Pattern

The repository companion object should define the Magnum repository instance and provide layers for dependency injection:

```scala
object PostgreSQLEntityRepository:
    // Create Repo with separate CreateDTO and EntityDTO types
    // CreateDTO is used for insertions, EntityDTO for reading and updating
    val repo = Repo[CreateEntityDTO, EntityDTO, Long]

    // Layer that provides repository implementation using PostgreSQLTransactor
    val layer: ZLayer[PostgreSQLTransactor, Nothing, EntityRepository] =
        ZLayer.fromFunction { (transactor: PostgreSQLTransactor) =>
            new PostgreSQLEntityRepository(transactor.transactor)
        }
end PostgreSQLEntityRepository
```

Note how we specify both the creation DTO and the entity DTO in the `Repo` definition, which enables proper type-safe operations for both insertions and queries.

## Schema Naming with SqlName Annotation

Use the `@SqlName` annotation to explicitly name database tables rather than relying on automatic naming:

```scala
// Explicitly name the table "xml_snapshots"
@SqlName("xml_snapshots")
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
case class XmlSnapshotDTO(
    @Id id: Long,
    content: String,
    // ... other fields
)
```

This practice makes it clear which table the DTO maps to and prevents naming discrepancies.

## Integration Testing Patterns

### Test Structure

Organize your integration tests with clear, descriptive test names and proper layer setup:

```scala
def spec = (suite("PostgreSQLEntityRepository")(
    test("should save and retrieve an entity by ID") {
        for
            repository <- ZIO.service[EntityRepository]
            entity = CreateEntity("test-entity", Some("description"))
            id <- repository.save(entity)
            retrieved <- repository.getById(id)
        yield assertTrue(
            retrieved.name == "test-entity",
            retrieved.description.contains("description")
        )
    },
    // More tests...
) @@ TestAspect.sequential @@ TestAspect.withLiveClock @@ MigrateAspects.migrate).provideSomeShared[
    Scope
](
    PostgreSQLTestingLayers.flywayMigrationServiceLayer,
    PostgreSQLEntityRepository.layer
)
```

### Test Assertions

Use ZIO Test assertions for cleaner and more expressive tests:

```scala
// For option values, use is matcher with_.some helper
assertTrue(
    latest.is(_.some).content == "expected content",
    latest.is(_.some).id == expectedId
)
```

## Layer Organization with PostgreSQL Infrastructure

When using the project's PostgreSQL infrastructure components, use the following patterns:

### Repository Layer Definition

```scala
// Define a layer that requires PostgreSQLTransactor
val layer: ZLayer[PostgreSQLTransactor, Nothing, YourRepository] =
    ZLayer.fromFunction { (transactor: PostgreSQLTransactor) =>
        new PostgreSQLYourRepository(transactor.transactor)
    }
```

### Test Specific Layers

For testing, use the test infrastructure and migration aspects:

```scala
// In test specification:
yourSpec
    .provideSomeShared[Scope](
        PostgreSQLTestingLayers.flywayMigrationServiceLayer,
        YourRepository.layer
    ) @@ TestAspect.sequential @@ TestAspect.withLiveClock @@ MigrateAspects.migrate
```

This ensures proper database setup and teardown for tests.

## Common Pitfalls

1. **N+1 Query Problem**:
   - Don't query related entities in loops
   - Use joins or batch loading with Magnum's batching capabilities

2. **Ignoring CQRS Principles**:
   - Remember that write operations should return minimal data
   - Keep reads separate from writes

3. **Not Using Transactor Correctly**:
   - Use `xa.connect` for read-only operations
   - Use `xa.transact` for write operations that need transactions

4. **Raw SQL Injection**:
   - Always use Magnum's `sql` interpolator for safe query building
   - Never concatenate strings for SQL

5. **Improper Error Handling**:
   - Use domain-specific errors that are meaningful to the application
   - Map infrastructure errors to domain errors when appropriate
   - Add `.orDie` for infrastructure errors that shouldn't propagate as domain errors
   - Consider providing detailed error messages for domain errors

6. **Missing Repository Tests**:
   - Always test repositories with real databases using TestContainers
   - Test both happy path and error scenarios
   - Test domain-specific error handling thoroughly

7. **Not Using DTOs Correctly**:
   - Keep clear separation between domain models and database DTOs
   - Use consistent transformation patterns

8. **Missing Database Indexes**:
   - Add indexes for commonly queried fields
   - Consider adding indexes for foreign keys

9. **Leaking Persistence Concerns to Domain**:
   - Repository interfaces should use domain types, not persistence types
   - Domain shouldn't need to know about database-specific constraints

10. **Using Generic Error Types**:
    - Avoid using `Nothing` as error type unless truly nothing can go wrong
    - Define meaningful domain errors for each failure case

11. **Mishandling Optional Results**:
    - Always handle `Option` returns from repository methods properly
    - Convert to domain errors for clear error messages

12. **SQL String Interpolation Issues**:
    - Be careful with string interpolation in SQL queries
    - Use `sql"column = $value"` for parameterized queries, not string concatenation

13. **Incorrect Magnum Method Chaining**:
    - Ensure methods like `map`, `filter`, etc. are called on the right objects
    - Keep transformations within the database operation block where possible

## Checklist

When implementing a repository, ensure:

- [ ] Repository implements the domain-defined repository interface
- [ ] Repository interface uses domain types and domain-relevant error types
- [ ] DTOs are defined with proper table and column mappings
- [ ] Conversions between domain models and DTOs are implemented
- [ ] Error handling strategy maps infrastructure errors to domain errors
- [ ] Transaction boundaries are properly defined
- [ ] Dynamic queries use the Spec pattern for flexibility
- [ ] Integration tests cover all repository methods and error cases
- [ ] CQRS principles are followed (commands don't return unnecessary data)
- [ ] Performance considerations (batch operations, indexes) are addressed
- [ ] No domain logic mixed with data access logic
