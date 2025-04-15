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

Our repository design is based on composable traits with clear responsibilities:

```scala
GenericReadRepository
  ├── GenericLoadService
  ├── GenericLoadAllService
  └── GenericFindService
  
GenericWriteRepository
  └── save(key, value): Op[Unit]
  
CreateRepository
  └── create(value): Op[Key]
```

### Key Repository Types

#### Read Operations

- **LoadRepository**: Load a single entity by key
- **ReadRepository**: Complete read operations (load/loadAll/find)
- **UpdateNotifyRepository**: Stream of entity updates

#### Write Operations

- **WriteRepository**: Update existing entities
- **CreateRepository**: Create new entities with generated keys
- **WriteRepositoryWithKeyAssignment**: Variant that handles key generation

#### Combined Repositories

- **Repository**: Standard read/write operations
- **RepositoryWithCreate**: Standard repository with creation capability
- **RepositoryWithKeyAssignment**: Repository with key generation

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

## Transactor Configuration

The correct way to create a ZLayer for the Transactor is to compose it with your DataSource layer:

```scala
// CORRECT APPROACH - Use ZLayer.service + flatMap
val transactorLayer: ZLayer[DataSource & Scope, Throwable, Transactor] =
  ZLayer.service[DataSource].flatMap { env => 
      Transactor.layer(env.get[DataSource])
  }

// Repository layer organization
val repositoryLayer: ZLayer[Transactor, Nothing, TransactionRepository] =
    ZLayer.fromFunction { (xa: Transactor) =>
        PostgreSQLTransactionRepository(xa)
    }

// Full layer including all dependencies
val fullLayer: ZLayer[Scope, Throwable, TransactionRepository] =
    dataSourceLayer >>>
        transactorLayer >>>
        repositoryLayer
```

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

Repository Implementations require thorough integration testing:

### Integration Testing with TestContainers

```scala
import org.scalatest.funsuite.AsyncFunSuite
import org.scalatest.matchers.should.Matchers
import org.scalatest.BeforeAndAfterAll
import org.testcontainers.containers.PostgreSQLContainer
import zio.{Runtime, Unsafe, ZIO}
import com.augustnagro.magnum.*
import com.augustnagro.magnum.magzio.*

class PostgreSQLAccountRepositorySpec extends AsyncFunSuite with Matchers with BeforeAndAfterAll:
  // Start PostgreSQL container
  val postgres = new PostgreSQLContainer("postgres:14")
  postgres.start()
  
  // Set up data source
  val dataSource: javax.sql.DataSource = {
    val ds = new com.zaxxer.hikari.HikariDataSource()
    ds.setJdbcUrl(postgres.getJdbcUrl)
    ds.setUsername(postgres.getUsername)
    ds.setPassword(postgres.getPassword)
    ds
  }
  
  // Set up transactor
  val xa = Transactor(dataSource)
  
  // Run migrations to set up schema
  val flyway = Flyway.configure()
    .dataSource(dataSource)
    .locations("classpath:db/migration")
    .load()
  
  flyway.migrate()
  
  // Create repository
  val repository = new PostgreSQLAccountRepository(xa)
  
  // Helper to run ZIO effects in tests
  def unsafeRun[E, A](zio: ZIO[Any, E, A]): A =
    Unsafe.unsafe { implicit unsafe =>
      Runtime.default.unsafe.run(zio).getOrThrowFiberFailure()
    }
  
  // Clean up after tests
  override def afterAll(): Unit =
    dataSource.asInstanceOf[com.zaxxer.hikari.HikariDataSource].close()
    postgres.stop()
  
  // Tests
  test("save should insert a new account") {
    // Test implementation
  }
  
  // Additional tests...
```

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