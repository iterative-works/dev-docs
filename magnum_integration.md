---
status: draft
last updated: 2025-04-06
version: "0.1"
---
> [!draft] Draft Document
> This document is an initial draft and may change significantly.
# Magnum Library Integration Guidelines

## Overview

This document outlines best practices for integrating the Magnum database access library with our repository pattern in the YNAB Importer project. Magnum provides a type-safe and efficient SQL interface that works with our ZIO-based architecture.

## Repository Types and Usage

Magnum provides two main types of repositories:

### 1. ImmutableRepo - Read-Only Operations

`ImmutableRepo` provides read-only access to the database and is used when you don't need to modify data:

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

### 2. Repo - Full CRUD Operations

`Repo` extends `ImmutableRepo` and adds write operations:

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

### Working with Repositories

A typical workflow using repositories:

```scala
// In service layer
class UserService(xa: Transactor):
  import UserRepo.repo
  
  def getUserById(id: Long): UIO[Option[User]] =
    xa.connect:
      repo.findById(id).map(_.toDomain)
    .orDie
    
  def createUser(newUser: UserCreator): UIO[User] =
    xa.transact:
      repo.insertReturning(newUser)
    .orDie
```

### Entity Creation Pattern with Generated IDs

When working with auto-generated IDs (like database sequences), use a separate Creator class:

```scala
// Entity Creator (no ID field)
case class UserCreator(
  name: String,
  email: String
) derives DbCodec

// Full Entity
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
case class User(
  @Id id: Long,
  name: String,
  email: String
) derives DbCodec

// Repository definition
val userRepo = Repo[UserCreator, User, Long]

// Usage
def createUser(name: String, email: String): UIO[Long] =
  xa.transact:
    userRepo.insertReturning(UserCreator(name, email)).id
  .orDie
```

## Data Transfer Objects (DTOs)

We use DTOs to separate our domain models from the database representation. We usually use chimney library to make the translations between DTOs and domain model easier:

1. Create a DTO class with the `@Table` annotation to define the database mapping:

```scala
@SqlName("transaction")
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
case class TransactionDTO(
    // Core identity fields
    sourceAccountId: Long,
    transactionId: String,
    
    // Other data fields
    date: LocalDate,
    amount: BigDecimal,
    // ...
) derives DbCodec
```

2. Provide transformation methods between DTOs and domain models:

```scala
// Inside the DTO class
def toTransaction: Transaction = this.into[Transaction]
    .withFieldComputed(
        _.id,
        dto => TransactionId(dto.sourceAccountId, dto.transactionId)
    )
    .transform

// In a companion object
object Transaction:
    def fromModel(model: Transaction): TransactionDTO =
        model.into[TransactionDTO]
            .withFieldComputed(_.sourceAccountId, _.id.sourceAccountId)
            .withFieldComputed(_.transactionId, _.id.transactionId)
            .transform
```

## ZIO Integration

Magnum provides ZIO integration through the `magnumzio` module:

1. Import the necessary packages:
```scala
import com.augustnagro.magnum.magzio.*
```

2. Use the transactor with ZIO effects:
```scala
// Within a repository method
override def find(filter: SomeQuery): UIO[Seq[Entity]] =
    xa.transact:
        // Use Magnum's Spec for dynamic queries
        val spec = Spec[EntityDTO]
            .where(filter.id.map(id => sql"id = $id").getOrElse(sql""))
            // ...other conditions
        
        // Convert DTOs to domain models
        entityRepo.findAll(spec).map(_.toDomainModel)
    .orDie
```

## Transactor Configuration with DataSource

The correct way to create a ZLayer for the Transactor is to compose it with your DataSource layer:

```scala
// CORRECT APPROACH - Use ZLayer.service + flatMap
val transactorLayer: ZLayer[DataSource & Scope, Throwable, Transactor] =
  ZLayer.service[DataSource].flatMap { env => 
      Transactor.layer(env.get[DataSource])
  }
```

This approach ensures that the DataSource is properly acquired and released as part of the ZIO Resource management lifecycle.

## Repository Layer Organization

Organize your repository layers in a hierarchical pattern:

```scala
// Repository implementation layer
val layer: ZLayer[Transactor, Nothing, TransactionRepository] =
    ZLayer.fromFunction { (xa: Transactor) =>
        PostgreSQLTransactionRepository(xa)
    }

// Full layer including all dependencies
val fullLayer: ZLayer[Scope, Throwable, TransactionRepository] =
    dataSourceLayer >>>
        transactorLayer >>>
        layer
```

## Custom Type Codecs

Define DbCodec instances for custom types or when the default mapping doesn't suit your needs:

```scala
// For LocalDate
given DbCodec[LocalDate] = DbCodec.SqlDateCodec.biMap(
    d => d.toLocalDate,
    d => java.sql.Date.valueOf(d)
)

// For Instant
given DbCodec[Instant] = DbCodec.SqlTimestampCodec.biMap(
    i => i.toInstant,
    i => java.sql.Timestamp.from(i)
)
```

## Dynamic Queries with Spec

Use Magnum's `Spec` class for building dynamic queries with multiple optional filter conditions:

```scala
val spec = Spec[EntityDTO]
    .where(
        filter.fieldOpt.map(value =>
            sql"field = $value"
        ).getOrElse(sql"")
    )
    .where(
        filter.dateFrom.map(date =>
            sql"created_at >= ${java.sql.Date.valueOf(date)}"
        ).getOrElse(sql"")
    )
    .limit(pageSize)
    .offset(pageNum * pageSize)
```

## Best Practices

1. Keep DTOs in the infrastructure layer, domain models in the core
2. Use `deriving DbCodec` for automatic codec generation
3. Properly handle SQL exceptions using `.orDie` or more granular error handling
4. Use `xa.connect` for read operations and `xa.transact` for write operations
5. Leverage Magnum's SQL interpolator for type-safe queries
6. Define DTOs as case classes with clear field naming matching the database schema
7. Use the repository pattern consistently throughout the application