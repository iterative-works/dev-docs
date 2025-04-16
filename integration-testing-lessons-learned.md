---
status: draft
created: 2025-04-16
last_updated: 2025-04-16
version: 0.1
tags:
  - zio
  - testing
  - scala
  - functional-programming
---
# Integration Testing Lessons Learned

## Overview

This document summarizes key lessons learned from implementing the `NormaXmlExportServiceSpec` integration test. It highlights important patterns, common pitfalls, and best practices to follow when creating integration tests in our ZIO-based codebase.

## Key Lessons

### 1. Layer Composition and Ordering

**Lesson:** The order of layer provision and test aspects is critical. Some aspects rely on layers being available, so we need to apply aspects first, then provide layers.

**Correct Pattern:**
```scala
yourSpec
  @@ MigrateAspects.migrate @@ TestAspect.sequential
  .provideSome[Scope](
    YourLayers.here
  )
```

**Explanation:**
- Applying aspects like `MigrateAspects.migrate` first ensures they have access to all layers provided later
- Using `.provideSome[Scope]` allows the test to inherit the `Scope` from the ZIO testing environment

### 2. Automatic Layer Composition

**Lesson:** ZIO's automatic layer composition with `provideSome` is generally simpler and less error-prone than manual layer composition.

**Prefer:**
```scala
.provideSome[Scope](
  PostgreSQLXmlSnapshotRepository.layer,
  XmlSnapshotService.layer,
  NormaService.layer,
  NormaXmlExportService.layer,
  normaRepoLayer,
  PostgreSQLTestingLayers.flywayMigrationServiceLayer
)
```

**Instead of:**
```scala
// Manual composition is more complex and error-prone
private val fullLayer = serviceLayer ++ normaRepoLayer ++ xmlSnapshotRepoLayer

.provide(
  ZLayer.make[NormaXmlExportService & NormaRepository & XmlSnapshotRepository](
    fullLayer,
    Scope.default
  ),
  createTablesLayer
)
```

**Explanation:**
- ZIO automatically resolves dependencies between layers
- Less code and less opportunity for errors
- Clearer to understand which services are being provided

### 3. Import Conflicts and Ambiguity

**Lesson:** Be careful with wildcard imports, especially from libraries like Magnum, to avoid ambiguous references.

**Avoid:**
```scala
import com.augustnagro.magnum.*
import com.augustnagro.magnum.magzio.*  // Introduces ambiguity for Transactor, sql, etc.
```

**Prefer:**
```scala
import com.augustnagro.magnum.magzio.*
// Import specific symbols from com.augustnagro.magnum as needed
import com.augustnagro.magnum.{Spec, execute} // Explicit imports
```

**Explanation:**
- Wildcard imports can cause ambiguous references (e.g., Transactor from both packages)
- Explicit imports make it clear which version is being used

### 4. SQL String Execution

**Lesson:** When executing raw SQL strings, use the proper methods from Magnum.

**Correct:**
```scala
com.augustnagro.magnum.execute(sqlString).update
```

**Instead of:**
```scala
sql(sqlString).update.run  // Incorrect - sql is for string interpolation
```

**Explanation:**
- `sql` is for string interpolation with parameters
- `execute` is for executing raw SQL strings

### 5. Type Compatibility in Assertions

**Lesson:** Be aware of database field types vs. domain types, especially for booleans vs. integers.

**Original issue:**
```scala
xmlSnapshot.exists(_.published == 1)  // Error: comparing Boolean and Int
```

**Fixed approach:**
```scala
xmlSnapshot.exists(_.published)  // If published is a Boolean
```

**Explanation:**
- Database representation (smallint 0/1) may differ from domain representation (Boolean)
- Just use the corresponding domain model check

### 6. Schema Migration Strategy

**Lesson:** Use Flyway for migrations instead of manual table creation in tests.

**Prefer:**
- Use existing migration files in `db/migration` directory
- Apply them with `MigrateAspects.migrate`

**Instead of:**
```scala
private val createTablesSql = """CREATE TABLE IF NOT EXISTS..."""
```

**Explanation:**
- Flyway provides consistent schema management
- Reduces duplication between test code and actual migrations
- Ensures tests run against the same schema as production

### 7. Test Container Setup

**Lesson:** Leverage the existing PostgreSQLTestingLayers for standardized test database setup.

**Pattern to follow:**
```scala
PostgreSQLTestingLayers.flywayMigrationServiceLayer
```

**Explanation:**
- Provides a consistent way to set up test databases
- Handles container lifecycle and configuration
- Integrates with Flyway for migrations

### 8. Clear Test Naming and Structure

**Lesson:** Use descriptive test names and structure tests to clearly indicate what's being tested.

**Pattern to follow:**
```scala
test("should create and retrieve a complete snapshot") {
  for
    // Arrange
    service <- ZIO.service[NormaXmlExportService]
    testLabel = s"integration-test-${currentTime}"

    // Act
    snapshotId <- service.takeCompleteSnapshot(...)
    xmlSnapshot <- ZIO.serviceWithZIO[XmlSnapshotRepository](_.findById(snapshotId))

    // Assert
    yield assertTrue(...)
}
```

**Explanation:**
- Clear test name describes the behavior being tested
- Arrange-Act-Assert pattern makes test structure clear
- Comments help separate sections when needed

### 9. Aspect and layer composition order

**Lesson:** The order of `provide` methods and TestAspect composition with `@@` does matter as soon as the aspect itself depends on some services.

**Original issue:**

```scala
...
) @@ MigrateAspects.migrate @@ TestAspect.sequential @@ TestAspect.withLiveEnv).provideSome[Scope](
    // db layers needing environment
) // Failing with None.get, because the live environment is not available to the layers
```

**Fixed approach:**

```scala
...
) @@ MigrateAspects.migrate @@ TestAspect.sequential).provideSome[Scope](
    // db layers needing environment
) @@ TestAspect.withLiveEnv // Working, the layers are now included in the live env
```

**Explanation:**
- The aspects wrap the preceding effect and change some aspect, like access to services etc. Therefore they need to wrap the complete effect they need to update.


## Common Integration Testing Pitfalls

1. **Not Cleaning Database Between Tests**
   - Always apply `MigrateAspects.migrate` to ensure a clean database state
   - Use `TestAspect.sequential` to prevent race conditions

2. **Hardcoded Test Data**
   - Use dynamic test data generation with timestamps to avoid collisions
   - Example: `s"integration-test-${currentTime}"`

3. **Not Validating Real Integration Points**
   - Ensure tests validate the actual integration behavior
   - Example: Create data with one service, verify it with another

4. **Complex Manual Layer Construction**
   - Leverage ZIO's automatic layer resolution with `provideSome`
   - Minimize manual layer combination with `>>>` and `++`

5. **Unnecessary Test Setup Duplication**
   - Use shared layers and test utilities
   - Leverage test aspect composition

## Best Practices for Future Integration Tests

1. **Layer Structure Pattern**
   ```scala
   // Repository/service under test
   private val repoLayer = ...

   // Test suite
   def spec = suite("Integration Test")(
     test("should do something") { ... }
   )
   @@ MigrateAspects.migrate
   @@ TestAspect.sequential
   .provideSome[Scope](
     // Provide only what's needed
     PostgreSQLTestingLayers.flywayMigrationServiceLayer,
     repoLayer,
     otherRequiredLayers
   )
   ```

2. **Database Container Pattern**
   - Always use the standard `PostgreSQLTestingLayers` for test containers
   - Let Flyway handle schema migrations via `MigrateAspects.migrate`

3. **Test Isolation Pattern**
   - Use unique identifiers with timestamps for test data
   - Apply `TestAspect.sequential` to prevent test interference

4. **Service Access Pattern**
   - Use `ZIO.service[YourService]` to access services
   - Use `ZIO.serviceWithZIO[YourRepository](_.method(...))` for repository calls

5. **Assertion Pattern**
   - Use `assertTrue(...)` with multiple conditions for clear assertions
   - Avoid custom matchers unless they significantly improve readability

## Application to Future Tests

When implementing new integration tests:

1. Start with the proper ZIOSpecDefault extension and imports
2. Define necessary service/repository layers
3. Structure the test suite with descriptive test names
4. Apply the appropriate test aspects (migration, sequential execution)
5. Provide the necessary layers using provideSome
6. Implement tests with clear arrange-act-assert structure
7. Validate actual integration behavior across component boundaries

By following these patterns, integration tests will be more consistent, reliable, and maintainable.
