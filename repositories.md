---
status: draft
last updated: 2025-04-06
version: "0.1"
---
> [!draft] Draft Document
> This document is an initial draft and may change significantly.
# Repository Pattern Guidelines

## Core Repository Architecture

Our codebase follows a repository pattern based on CQRS (Command Query Responsibility Segregation) principles for data access. This document outlines the repository interfaces and usage patterns.

## Repository Traits Hierarchy

Our repository design is based on composable traits with clear responsibilities:

```
GenericReadRepository
  ├── GenericLoadService
  ├── GenericLoadAllService
  └── GenericFindService
  
GenericWriteRepository
  └── save(key, value): Op[Unit]
  
CreateRepository
  └── create(value): Op[Key]
```

## Key Repository Types

### Read Operations

- **LoadRepository**: Load a single entity by key
- **ReadRepository**: Complete read operations (load/loadAll/find)
- **UpdateNotifyRepository**: Stream of entity updates

### Write Operations

- **WriteRepository**: Update existing entities
- **CreateRepository**: Create new entities with generated keys
- **WriteRepositoryWithKeyAssignment**: Variant that handles key generation

### Combined Repositories

- **Repository**: Standard read/write operations
- **RepositoryWithCreate**: Standard repository with creation capability
- **RepositoryWithKeyAssignment**: Repository with key generation

## Working with Repositories

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
```

## CQRS Principles

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

## Implementation Notes

- Repository implementations should be in the infrastructure layer
- Domain models stay in the core without infrastructure dependencies
- Use ZIO environments to provide repository implementations
- Test repositories with integration tests for actual behavior