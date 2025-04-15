---
status: maintained
last updated: 2023-05-20
version: "1.0"
---

# Implementation Guide: Repository Implementation

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

## Repository Pattern in Our Project

Our codebase follows a repository pattern based on CQRS (Command Query Responsibility Segregation) principles for data access. We have a specific repository traits hierarchy to promote consistent implementation.

### Repository Traits Hierarchy

Our repository design is based on composable traits with clear responsibilities. The hierarchy is defined in our core library:

```scala
// Basic service traits
trait GenericLoadService[Eff[+_], -Key, +Value]:
    type Op[A] = Eff[A]
    def load(id: Key): Op[Option[Value]]

trait GenericUpdateNotifyService[Str[+_], Key]:
    def updates: Str[Key]

trait GenericLoadAllService[Eff[+_], Coll[+_], -Key, +Value]:
    type Op[A] = Eff[A]
    def loadAll(ids: Seq[Key]): Op[Coll[Value]]

trait GenericFindService[Eff[+_], Coll[+_], -Key, +Value, -FilterArg]:
    type Op[A] = Eff[A]
    def find(filter: FilterArg): Op[Coll[Value]]

// Read repository combines load, loadAll, and find operations
trait GenericReadRepository[Eff[+_], Coll[+_], -Key, +Value, -FilterArg]
    extends GenericLoadService[Eff, Key, Value]
    with GenericLoadAllService[Eff, Coll, Key, Value]
    with GenericFindService[Eff, Coll, Key, Value, FilterArg]

// Write repository for saving existing entities
trait GenericWriteRepository[Eff[_], -Key, -Value]:
    type Op[A] = Eff[A]
    def save(key: Key, value: Value): Op[Unit]

// Marker trait for creation DTOs
trait Create[A]

// Repository for creating new entities
trait GenericCreateRepository[Eff[+_], +Key, -Init]:
    type Op[A] = Eff[A]
    def create(value: Init): Op[Key]

// Write repository with key assignment
trait GenericWriteRepositoryWithKeyAssignment[Eff[+_], +Key, -Value]:
    type Op[A] = Eff[A]
    def save(value: Value): Op[Key]
    def save(value: Key => Value): Op[Key]

// Combined read and write repository
trait GenericRepository[Eff[+_], -Key, Value]
    extends GenericReadRepository[Eff, Seq, Key, Value, Unit]
    with GenericWriteRepository[Eff, Key, Value]
```

### Key Repository Types

Our framework provides specialized repository types for ZIO integration using `UIO` for operations:

#### Read Operations

```scala
// Basic load repository
type LoadRepository[-Key, +Value] = GenericLoadService[UIO, Key, Value]

// Load multiple entities by keys
type LoadAllRepository[-Key, +Value] = GenericLoadAllService[UIO, Seq, Key, Value]

// Complete read operations
trait ReadRepository[-Key, +Value, -FilterArg]
    extends GenericReadRepository[UIO, Seq, Key, Value, FilterArg]:
    override def loadAll(ids: Seq[Key]): UIO[Seq[Value]] =
        // Inefficient implementation, meant to be overridden
        ZIO.foreach(ids)(load).map(_.flatten)

// Repository for notifications on updates
trait UpdateNotifyRepository[Key]
    extends GenericUpdateNotifyService[UStream, Key]
```

#### Write Operations

```scala
// Basic write repository
trait WriteRepository[-Key, -Value]
    extends GenericWriteRepository[UIO, Key, Value]

// Repository for creating new entities
trait CreateRepository[+Key, -Init]
    extends GenericCreateRepository[UIO, Key, Init]

// Write repository with automatic key assignment
trait WriteRepositoryWithKeyAssignment[Key, -Value]
    extends GenericWriteRepositoryWithKeyAssignment[UIO, Key, Value]
```

#### Combined Repositories

```scala
// Standard read/write repository
trait Repository[-Key, Value, -FilterArg]
    extends ReadRepository[Key, Value, FilterArg]
    with WriteRepository[Key, Value]

// Repository with creation capability
trait RepositoryWithCreate[Key, Value, -FilterArg, -Init <: Create[Value]]
    extends ReadRepository[Key, Value, FilterArg]
    with WriteRepository[Key, Value]
    with CreateRepository[Key, Init]

// Repository with key generation
trait RepositoryWithKeyAssignment[Key, Value, -FilterArg]
    extends ReadRepository[Key, Value, FilterArg]
    with WriteRepositoryWithKeyAssignment[Key, Value]
```

### CQRS Principles

Our repository design follows these CQRS principles:

1. **Commands Don't Return Domain Data**
   - `save` returns `Unit`, not the entity
   - Maintains separation between commands and queries

2. **Creation Returns Only Keys**
   - `create` returns the generated key/ID
   - Requires explicit query to retrieve the created entity

3. **Queries Don't Modify State**
   - `load`, `find` and other read methods are pure
   - No side effects in query operations

4. **Event Notification via Streams**
   - `updates` stream for reactive patterns
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

### Basic Structure with Magnum

```scala
import com.augustnagro.magnum.*
import com.augustnagro.magnum.magzio.*
import zio.*

class PostgreSQLUserRepository(xa: Transactor) extends UserRepository:
  import UserDTO.given
  import UserRepo.repo

  override def findById(id: UserId): UIO[Option[User]] =
    xa.connect:
      repo.findById(id.value).map(_.map(_.toDomain))
    .orDie
  
  override def findByEmail(email: EmailAddress): UIO[Option[User]] =
    xa.connect:
      // Use Magnum's Spec for dynamic queries
      val spec = Spec[UserDTO].where(sql"email = ${email.value}")
      repo.findAll(spec).headOption.map(_.map(_.toDomain))
    .orDie
  
  override def findAll: UIO[Seq[User]] =
    xa.connect:
      repo.findAll.map(_.map(_.toDomain).toSeq)
    .orDie
  
  override def save(user: User): UIO[Unit] =
    xa.transact:
      repo.update(UserDTO.fromDomain(user))
    .orDie
  
  override def create(user: CreateUser): UIO[UserId] =
    xa.transact:
      val dto = UserDTO.fromCreate(user)
      repo.insertReturning(dto).map(created => UserId(created.id))
    .orDie
  
  override def delete(id: UserId): UIO[Unit] =
    xa.transact:
      repo.deleteById(id.value)
    .orDie

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
  def fromCreate(create: CreateUser): UserDTO =
    UserDTO(
      id = 0, // Will be generated by database
      email = create.email.value,
      name = create.name,
      hashedPassword = create.hashedPassword,
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
case class SourceAccount(id: Long, accountId: String, /* other fields */)

// Creation DTO
case class CreateSourceAccount(accountId: String, /* other fields - no ID */)
  extends Create[SourceAccount]

// In service layer
def createNewAccount(account: CreateSourceAccount): UIO[Long] =
  for
    id <- repository.create(account)
    _ <- logService.info(s"Created account with ID: $id")
  yield id

// If you need the created entity
def createAndRetrieveAccount(account: CreateSourceAccount): UIO[SourceAccount] =
  for
    id <- repository.create(account)
    entityOpt <- repository.load(id)
    entity <- ZIO.fromOption(entityOpt).orElseFail(new RuntimeException(s"Created entity not found: $id"))
  yield entity
```

### Entity Update Pattern

```scala
// In service layer
def updateAccount(id: Long, account: SourceAccount): UIO[Unit] =
  repository.save(id, account)

// When you need to verify entity exists first
def safeUpdateAccount(id: Long, account: SourceAccount): UIO[Unit] =
  for
    existing <- repository.load(id)
    _ <- ZIO.fromOption(existing).flatMap(_ => repository.save(id, account))
      .orElseFail(new RuntimeException(s"Entity not found: $id"))
  yield ()
```

### Query Pattern

```scala
// Domain query model
case class AccountQuery(active: Option[Boolean] = None, /* other filters */)

// In service layer
def findActiveAccounts(): UIO[Seq[SourceAccount]] =
  repository.find(AccountQuery(active = Some(true)))

// Implementation using Magnum Spec
override def find(query: AccountQuery): UIO[Seq[SourceAccount]] =
  xa.connect:
    // Build dynamic query with Magnum's Spec
    val spec = Spec[AccountDTO]
      .where(query.active.map(active => 
        sql"status = ${if (active) "ACTIVE" else "INACTIVE"}"
      ).getOrElse(sql""))
      // Add other conditions as needed
    
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
object PostgreSQLTransactionRepository:
    // Repository layer taking only the PostgreSQLTransactor dependency
    val layer: ZLayer[PostgreSQLTransactor, Nothing, TransactionRepository] =
        ZLayer.fromFunction { (ts: PostgreSQLTransactor) =>
            PostgreSQLTransactionRepository(ts.transactor)
        }

    // Full layer including all dependencies from scratch
    val fullLayer: ZLayer[Scope, Throwable, TransactionRepository] =
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
val repositoryLayerWithMigrations: ZLayer[Scope, Throwable, TransactionRepository] =
    PostgreSQLDatabaseSupport.layerWithMigrations() >>> 
        PostgreSQLTransactionRepository.layer
        
// To specify additional migration locations
val repositoryLayerWithCustomMigrations: ZLayer[Scope, Throwable, TransactionRepository] =
    PostgreSQLDatabaseSupport.layerWithMigrations(
        additionalLocations = List("classpath:db/specific-migrations")
    ) >>> PostgreSQLTransactionRepository.layer
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
override def find(filter: TransactionQuery): UIO[Seq[Transaction]] =
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

object PostgreSQLTransactionRepositorySpec extends ZIOSpecDefault:
    // Define layers needed for repository tests
    val repositoryLayer =
        PostgreSQLTransactionRepository.layer ++
        PostgreSQLSourceAccountRepository.layer
  
    def spec = {
        suite("PostgreSQLTransactionRepository")(
            test("should save and retrieve a transaction") {
                for
                    // Get repository services
                    repo <- ZIO.service[TransactionRepository]
                    
                    // Create test data
                    transaction = createSampleTransaction
                    
                    // Execute operations
                    _ <- repo.save(transaction.id, transaction)
                    retrieved <- repo.load(transaction.id)
                yield
                    // Assert results
                    assertTrue(
                        retrieved.isDefined,
                        retrieved.get.id == transaction.id
                    )
            },
            
            test("should find transactions by filter") {
                for
                    repo <- ZIO.service[TransactionRepository]
                    
                    // Create test data
                    tx1 = createSampleTransaction
                    tx2 = tx1.copy(id = TransactionId("TX2"), amount = 200.0)
                    
                    // Save test data
                    _ <- repo.save(tx1.id, tx1)
                    _ <- repo.save(tx2.id, tx2)
                    
                    // Query with filter
                    results <- repo.find(TransactionQuery(
                        amountMin = Some(150.0)
                    ))
                yield
                    assertTrue(
                        results.size == 1,
                        results.head.id == tx2.id
                    )
            }
        // Apply aspects for all tests
        ) @@ sequential @@ migrate
    }.provideSomeShared[Scope](
        // Provide infrastructure layers
        flywayMigrationServiceLayer,
        repositoryLayer
    )
end PostgreSQLTransactionRepositorySpec
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
6. **Error Handling** - Test error scenarios like constraint violations

This approach ensures that repository implementations are thoroughly tested against real PostgreSQL databases with proper schema migrations, while keeping the tests maintainable and reliable.

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
   - Decide on consistent error handling strategy (.orDie vs specific error mapping)
   - Consider providing meaningful error messages for domain errors

6. **Missing Repository Tests**:
   - Always test repositories with real databases using TestContainers
   - Test both happy path and error scenarios

7. **Not Using DTOs Correctly**:
   - Keep clear separation between domain models and database DTOs
   - Use consistent transformation patterns

8. **Missing Database Indexes**:
   - Add indexes for commonly queried fields
   - Consider adding indexes for foreign keys

## Checklist

When implementing a repository, ensure:

- [ ] Repository implements the correct repository interface from the domain
- [ ] DTOs are defined with proper table and column mappings
- [ ] Conversions between domain models and DTOs are implemented
- [ ] Error handling strategy is consistent
- [ ] Transaction boundaries are properly defined
- [ ] Dynamic queries use the Spec pattern for flexibility
- [ ] Integration tests cover all repository methods
- [ ] CQRS principles are followed (commands don't return data)
- [ ] Performance considerations (batch operations, indexes) are addressed
- [ ] No domain logic mixed with data access logic