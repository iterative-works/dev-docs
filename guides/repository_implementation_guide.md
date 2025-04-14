---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
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

## Structure and Patterns

### Basic Structure

```scala
// PostgreSQL implementation of a repository interface
class PostgreSQLUserRepository(
  // Database connection/client
  dataSource: DataSource,
  // Helpers for DB operations
  dbExecutor: DBExecutor
) extends UserRepository:

  // Import repository error types defined in the domain
  import RepositoryError._

  /**
   * Find a user by ID.
   */
  override def findById(id: UserId): ZIO[Any, RepositoryError, Option[User]] =
    // Build SQL query
    val query = sql"""
      SELECT id, email, name, hashed_password, role, status, created_at, updated_at
      FROM users
      WHERE id = $id
    """
    
    // Execute query and map result
    dbExecutor.querySingle(query)(resultSetToUser)
      .mapError(e => DatabaseError(e))
  
  /**
   * Find a user by email address.
   */
  override def findByEmail(email: EmailAddress): ZIO[Any, RepositoryError, Option[User]] =
    val query = sql"""
      SELECT id, email, name, hashed_password, role, status, created_at, updated_at
      FROM users
      WHERE email = ${email.value}
    """
    
    dbExecutor.querySingle(query)(resultSetToUser)
      .mapError(e => DatabaseError(e))
  
  /**
   * Find all users.
   */
  override def findAll: ZIO[Any, RepositoryError, Seq[User]] =
    val query = sql"""
      SELECT id, email, name, hashed_password, role, status, created_at, updated_at
      FROM users
      ORDER BY created_at DESC
    """
    
    dbExecutor.queryList(query)(resultSetToUser)
      .mapError(e => DatabaseError(e))
  
  /**
   * Save a user (insert or update).
   */
  override def save(user: User): ZIO[Any, RepositoryError, User] =
    // Check if user exists
    findById(user.id).flatMap {
      case Some(_) => update(user)
      case None => insert(user)
    }
  
  /**
   * Delete a user by ID.
   */
  override def delete(id: UserId): ZIO[Any, RepositoryError, Unit] =
    val query = sql"""
      DELETE FROM users
      WHERE id = $id
    """
    
    dbExecutor.execute(query)
      .mapError(e => DatabaseError(e))
      .unit
  
  // Helper methods
  
  private def insert(user: User): ZIO[Any, RepositoryError, User] =
    val query = sql"""
      INSERT INTO users (id, email, name, hashed_password, role, status, created_at, updated_at)
      VALUES (
        ${user.id.value},
        ${user.email.value},
        ${user.name},
        ${user.hashedPassword},
        ${user.role.toString},
        ${user.status.toString},
        ${user.createdAt},
        ${user.updatedAt}
      )
    """
    
    dbExecutor.execute(query)
      .mapError {
        case e if isDuplicateKeyError(e) => DuplicateEntityError(s"User already exists with id ${user.id}")
        case e => DatabaseError(e)
      }
      .as(user)
  
  private def update(user: User): ZIO[Any, RepositoryError, User] =
    val query = sql"""
      UPDATE users
      SET email = ${user.email.value},
          name = ${user.name},
          hashed_password = ${user.hashedPassword},
          role = ${user.role.toString},
          status = ${user.status.toString},
          updated_at = ${user.updatedAt}
      WHERE id = ${user.id.value}
    """
    
    dbExecutor.execute(query)
      .mapError(e => DatabaseError(e))
      .as(user)
  
  private def resultSetToUser(rs: ResultSet): Option[User] =
    if !rs.next() then None
    else Some(
      User(
        id = UserId(UUID.fromString(rs.getString("id"))),
        email = EmailAddress.fromString(rs.getString("email")).getOrElse(
          throw new RuntimeException(s"Invalid email in database: ${rs.getString("email")}")  
        ),
        name = rs.getString("name"),
        hashedPassword = rs.getString("hashed_password"),
        role = UserRole.valueOf(rs.getString("role")),
        status = UserStatus.valueOf(rs.getString("status")),
        createdAt = rs.getTimestamp("created_at").toInstant,
        updatedAt = rs.getTimestamp("updated_at").toInstant
      )
    )
  
  private def isDuplicateKeyError(e: Throwable): Boolean =
    e.getMessage.contains("duplicate key")
```

### Key Structural Elements

1. **Class Declaration**:
   - Implement the corresponding repository interface from the domain layer
   - Accept database connectivity dependencies as constructor parameters
   - Name it based on technology and entity (`PostgreSQLUserRepository`)

2. **Method Implementations**:
   - Implement all methods declared in the interface
   - Translate between domain and database formats
   - Use SQL or ORM tooling appropriate for the database

3. **Query Methods**:
   - Write optimized queries for data retrieval
   - Map database results to domain entities
   - Handle error cases properly

4. **Save/Update Methods**:
   - Implement proper insert/update logic
   - Ensure data consistency
   - Handle common errors like duplicates

5. **Helper Methods**:
   - Extract common mapping logic to helper methods
   - Separate query construction from execution when complex

## Implementation Steps

1. **Choose Database Technology**:
   - Select an appropriate database for the entity
   - Consider query patterns, scalability, and team familiarity

2. **Create Database Schema**:
   - Design tables to store entity attributes
   - Create appropriate indexes for query performance
   - Set up migrations for schema changes

3. **Implement Core Methods**:
   - Start with basic CRUD operations
   - Focus on correct mapping between domain and database

4. **Add Domain-Specific Queries**:
   - Implement specialized query methods from the interface
   - Optimize for performance where needed

5. **Handle Error Cases**:
   - Map database errors to domain repository errors
   - Deal with constraint violations, connection issues, etc.

6. **Test Thoroughly**:
   - Create integration tests with actual database
   - Test edge cases and error conditions

## Examples

### Simple Example: PostgreSQLAccountRepository

```scala
import java.sql.{Connection, ResultSet}
import java.time.Instant
import java.util.UUID
import zio.ZIO
import javax.sql.DataSource

class PostgreSQLAccountRepository(
  dataSource: DataSource,
  dbExecutor: DBExecutor
) extends AccountRepository:

  override def findById(id: AccountId): ZIO[Any, RepositoryError, Option[Account]] =
    val query = sql"""
      SELECT a.id, a.customer_id, a.name, a.status, a.balance_amount, a.balance_currency, 
             a.created_at, a.updated_at
      FROM accounts a
      WHERE a.id = ${id.value}
    """
    
    dbExecutor.querySingle(query)(resultSetToAccount)
      .mapError(e => DatabaseError(e))

  override def findByCustomer(customerId: CustomerId): ZIO[Any, RepositoryError, Seq[Account]] =
    val query = sql"""
      SELECT a.id, a.customer_id, a.name, a.status, a.balance_amount, a.balance_currency, 
             a.created_at, a.updated_at
      FROM accounts a
      WHERE a.customer_id = ${customerId.value}
      ORDER BY a.created_at DESC
    """
    
    dbExecutor.queryList(query)(resultSetToAccount)
      .mapError(e => DatabaseError(e))
  
  override def findActive(): ZIO[Any, RepositoryError, Seq[Account]] =
    val query = sql"""
      SELECT a.id, a.customer_id, a.name, a.status, a.balance_amount, a.balance_currency, 
             a.created_at, a.updated_at
      FROM accounts a
      WHERE a.status = 'Active'
      ORDER BY a.created_at DESC
    """
    
    dbExecutor.queryList(query)(resultSetToAccount)
      .mapError(e => DatabaseError(e))
  
  override def save(account: Account): ZIO[Any, RepositoryError, Account] =
    findById(account.id).flatMap {
      case Some(_) => update(account)
      case None => insert(account)
    }
  
  override def delete(id: AccountId): ZIO[Any, RepositoryError, Unit] =
    val query = sql"""
      DELETE FROM accounts
      WHERE id = ${id.value}
    """
    
    dbExecutor.execute(query)
      .mapError(e => DatabaseError(e))
      .unit
  
  private def insert(account: Account): ZIO[Any, RepositoryError, Account] =
    val query = sql"""
      INSERT INTO accounts (
        id, customer_id, name, status, balance_amount, balance_currency, created_at, updated_at
      ) VALUES (
        ${account.id.value},
        ${account.customerId.value},
        ${account.name},
        ${account.status.toString},
        ${account.balance.amount},
        ${account.balance.currency.toString},
        ${account.createdAt},
        ${account.updatedAt}
      )
    """
    
    dbExecutor.execute(query)
      .mapError {
        case e if isDuplicateKeyError(e) => DuplicateEntityError(s"Account already exists with id ${account.id}")
        case e => DatabaseError(e)
      }
      .as(account)
  
  private def update(account: Account): ZIO[Any, RepositoryError, Account] =
    val query = sql"""
      UPDATE accounts
      SET customer_id = ${account.customerId.value},
          name = ${account.name},
          status = ${account.status.toString},
          balance_amount = ${account.balance.amount},
          balance_currency = ${account.balance.currency.toString},
          updated_at = ${account.updatedAt}
      WHERE id = ${account.id.value}
    """
    
    dbExecutor.execute(query)
      .mapError(e => DatabaseError(e))
      .as(account)
  
  private def resultSetToAccount(rs: ResultSet): Option[Account] =
    if !rs.next() then None
    else Some(
      Account(
        id = AccountId(UUID.fromString(rs.getString("id"))),
        customerId = CustomerId(UUID.fromString(rs.getString("customer_id"))),
        name = rs.getString("name"),
        status = AccountStatus.valueOf(rs.getString("status")),
        balance = Money(
          amount = rs.getBigDecimal("balance_amount"),
          currency = Currency.valueOf(rs.getString("balance_currency"))
        ),
        createdAt = rs.getTimestamp("created_at").toInstant,
        updatedAt = rs.getTimestamp("updated_at").toInstant
      )
    )
    
  private def isDuplicateKeyError(e: Throwable): Boolean =
    e.getMessage.contains("duplicate key")
```

### Complex Example: PostgreSQLTransactionRepository

```scala
class PostgreSQLTransactionRepository(
  dataSource: DataSource,
  dbExecutor: DBExecutor,
  clock: Clock
) extends TransactionRepository:

  // Implementing a complex query with pagination and multiple filters
  override def findByAccountAndDateRange(
    accountId: AccountId, 
    dateRange: DateRange
  ): ZIO[Any, RepositoryError, Seq[Transaction]] =
    val query = sql"""
      SELECT t.id, t.account_id, t.date, t.amount, t.currency,
             t.description, t.category, t.status, t.external_id, 
             t.created_at, t.updated_at
      FROM transactions t
      WHERE t.account_id = ${accountId.value}
        AND t.date >= ${dateRange.start}
        AND t.date <= ${dateRange.end}
      ORDER BY t.date DESC, t.created_at DESC
    """
    
    dbExecutor.queryList(query)(resultSetToTransaction)
      .mapError(e => DatabaseError(e))

  // Implementing method with pagination
  override def findByStatus(
    status: TransactionStatus, 
    limit: Int, 
    offset: Int
  ): ZIO[Any, RepositoryError, PaginatedResult[Transaction]] =
    for
      // Query for transactions with limit and offset
      transactions <- dbExecutor.queryList(
        sql"""
          SELECT t.id, t.account_id, t.date, t.amount, t.currency,
                 t.description, t.category, t.status, t.external_id, 
                 t.created_at, t.updated_at
          FROM transactions t
          WHERE t.status = ${status.toString}
          ORDER BY t.date DESC, t.created_at DESC
          LIMIT $limit OFFSET $offset
        """
      )(resultSetToTransaction).mapError(e => DatabaseError(e))
      
      // Count total matching records
      countResult <- dbExecutor.querySingle(
        sql"""
          SELECT COUNT(*) as total
          FROM transactions t
          WHERE t.status = ${status.toString}
        """
      )(rs => if rs.next() then Some(rs.getLong("total")) else None)
        .mapError(e => DatabaseError(e))
      
      totalCount = countResult.getOrElse(0L)
    yield PaginatedResult(
      items = transactions,
      totalCount = totalCount,
      pageNumber = offset / limit + 1,
      pageSize = limit,
      hasMore = totalCount > (offset + transactions.size)
    )

  // Implementing a complex grouping query
  override def sumByCategory(
    accountId: AccountId,
    categories: Set[TransactionCategory],
    dateRange: DateRange
  ): ZIO[Any, RepositoryError, Map[TransactionCategory, Money]] =
    // Convert categories to a list for the IN clause
    val categoryList = categories.map(_.toString).mkString("','") 
    
    val query = s"""
      SELECT t.category, t.currency, SUM(t.amount) as total
      FROM transactions t
      WHERE t.account_id = '${accountId.value}'
        AND t.date >= '${dateRange.start}'
        AND t.date <= '${dateRange.end}'
        AND t.category IN ('$categoryList')
      GROUP BY t.category, t.currency
    """
    
    dbExecutor.queryList(sql(query)) { rs =>
      if !rs.next() then None
      else Some(
        (
          TransactionCategory.valueOf(rs.getString("category")),
          Money(
            amount = rs.getBigDecimal("total"),
            currency = Currency.valueOf(rs.getString("currency"))
          )
        )
      )
    }.mapError(e => DatabaseError(e))
     .map(_.toMap)

  // Implementing batch operations
  override def saveAll(transactions: Seq[Transaction]): ZIO[Any, RepositoryError, Seq[Transaction]] =
    if transactions.isEmpty then
      ZIO.succeed(Seq.empty)
    else
      // Use batch insert for better performance
      val query = """
        INSERT INTO transactions (
          id, account_id, date, amount, currency,
          description, category, status, external_id, 
          created_at, updated_at
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ON CONFLICT (id) DO UPDATE SET
          account_id = EXCLUDED.account_id,
          date = EXCLUDED.date,
          amount = EXCLUDED.amount,
          currency = EXCLUDED.currency,
          description = EXCLUDED.description,
          category = EXCLUDED.category,
          status = EXCLUDED.status,
          external_id = EXCLUDED.external_id,
          updated_at = EXCLUDED.updated_at
      """
      
      // Execute with batch parameters
      ZIO.attemptBlocking {
        dataSource.getConnection.use { conn =>
          val stmt = conn.prepareStatement(query)
          
          transactions.foreach { tx =>
            var i = 1
            stmt.setObject(i, tx.id.value); i += 1
            stmt.setObject(i, tx.accountId.value); i += 1
            stmt.setDate(i, java.sql.Date.valueOf(tx.date.toLocalDate)); i += 1
            stmt.setBigDecimal(i, tx.amount.amount.bigDecimal); i += 1
            stmt.setString(i, tx.amount.currency.toString); i += 1
            stmt.setString(i, tx.description); i += 1
            stmt.setString(i, tx.category.map(_.toString).orNull); i += 1
            stmt.setString(i, tx.status.toString); i += 1
            stmt.setString(i, tx.externalId.orNull); i += 1
            stmt.setTimestamp(i, java.sql.Timestamp.from(tx.createdAt)); i += 1
            stmt.setTimestamp(i, java.sql.Timestamp.from(tx.updatedAt))
            
            stmt.addBatch()
          }
          
          stmt.executeBatch()
        }
      }.mapError(e => DatabaseError(e))
       .as(transactions)

  // Other methods omitted for brevity...
  
  private def resultSetToTransaction(rs: ResultSet): Option[Transaction] =
    if !rs.next() then None
    else Some(
      Transaction(
        id = TransactionId(UUID.fromString(rs.getString("id"))),
        accountId = AccountId(UUID.fromString(rs.getString("account_id"))),
        date = rs.getDate("date").toLocalDate,
        amount = Money(
          amount = rs.getBigDecimal("amount"),
          currency = Currency.valueOf(rs.getString("currency"))
        ),
        description = rs.getString("description"),
        category = Option(rs.getString("category")).map(TransactionCategory.valueOf),
        status = TransactionStatus.valueOf(rs.getString("status")),
        externalId = Option(rs.getString("external_id")),
        createdAt = rs.getTimestamp("created_at").toInstant,
        updatedAt = rs.getTimestamp("updated_at").toInstant
      )
    )
```

## Integration Points

Repository Implementations typically integrate with:

1. **Database Technologies**:
   - PostgreSQL, MySQL, MongoDB, etc.
   - Connection pools and clients

2. **Repository Interfaces**:
   - Implement interfaces defined in the domain layer
   - Follow the contract specified by the interface

3. **Domain Entities**:
   - Map database records to/from domain entities
   - Preserve entity invariants during conversion

4. **Error Handling**:
   - Map technical errors to domain-friendly RepositoryError types
   - Preserve error contexts for debugging

5. **Infrastructure Services**:
   - Connection pools
   - Transaction managers
   - Query builders

6. **Testing Infrastructure**:
   - Database migrations for test setup
   - Test containers for integration testing

## Testing Approach

Repository Implementations require thorough integration testing:

1. **Integration Testing with Test Containers**:
   - Test against real database instances in containers
   - Run migrations before tests to set up schema
   - Clean up data between tests

2. **CRUD Operation Tests**:
   - Test basic create, read, update, delete operations
   - Verify correct database state after operations

3. **Query Method Tests**:
   - Test complex queries with various parameters
   - Verify correct filtering, sorting, and pagination

4. **Error Handling Tests**:
   - Test behavior on constraint violations
   - Test recovery from connection issues
   - Verify correct error mapping

5. **Performance Tests**:
   - Test batch operations for efficiency
   - Verify query performance with realistic data volumes

### Example Integration Test Using TestContainers:

```scala
import org.scalatest.funsuite.AsyncFunSuite
import org.scalatest.matchers.should.Matchers
import org.scalatest.BeforeAndAfterAll
import org.testcontainers.containers.PostgreSQLContainer
import zio.{Runtime, Unsafe, ZIO}
import java.time.{Instant, LocalDate}
import java.util.UUID
import javax.sql.DataSource

class PostgreSQLAccountRepositorySpec extends AsyncFunSuite with Matchers with BeforeAndAfterAll:
  // Start PostgreSQL container
  val postgres = new PostgreSQLContainer("postgres:14")
  postgres.start()
  
  // Set up data source
  val dataSource: DataSource = {
    val ds = new com.zaxxer.hikari.HikariDataSource()
    ds.setJdbcUrl(postgres.getJdbcUrl)
    ds.setUsername(postgres.getUsername)
    ds.setPassword(postgres.getPassword)
    ds
  }
  
  // Set up database executor
  val dbExecutor = new JdbcExecutor(dataSource)
  
  // Run migrations to set up schema
  val flyway = Flyway.configure()
    .dataSource(dataSource)
    .locations("classpath:db/migration")
    .load()
  
  flyway.migrate()
  
  // Create repository
  val repository = new PostgreSQLAccountRepository(dataSource, dbExecutor)
  
  // Helper to run ZIO effects in tests
  def unsafeRun[E, A](zio: ZIO[Any, E, A]): A =
    Unsafe.unsafe { implicit unsafe =>
      Runtime.default.unsafe.run(zio).getOrThrowFiberFailure()
    }
  
  // Clean up after tests
  override def afterAll(): Unit =
    dataSource.asInstanceOf[com.zaxxer.hikari.HikariDataSource].close()
    postgres.stop()
  
  // Example test data
  val customerId = CustomerId(UUID.randomUUID)
  val accountId = AccountId(UUID.randomUUID)
  val now = Instant.now
  
  val testAccount = Account(
    id = accountId,
    customerId = customerId,
    name = "Test Account",
    status = AccountStatus.Active,
    balance = Money(100, Currency.USD),
    createdAt = now,
    updatedAt = now
  )
  
  // Tests
  test("save should insert a new account") {
    val result = unsafeRun(repository.save(testAccount))
    
    result shouldBe testAccount
    
    // Verify it's in the database
    val retrieved = unsafeRun(repository.findById(accountId))
    retrieved shouldBe Some(testAccount)
  }
  
  test("findByCustomer should return all customer accounts") {
    // Insert a second account for the same customer
    val secondAccountId = AccountId(UUID.randomUUID)
    val secondAccount = testAccount.copy(
      id = secondAccountId,
      name = "Second Account"
    )
    
    unsafeRun(repository.save(secondAccount))
    
    // Retrieve both accounts
    val accounts = unsafeRun(repository.findByCustomer(customerId))
    
    accounts should have size 2
    accounts.map(_.id) should contain allOf(accountId, secondAccountId)
  }
  
  test("findActive should return only active accounts") {
    // Insert an inactive account
    val inactiveAccountId = AccountId(UUID.randomUUID)
    val inactiveAccount = testAccount.copy(
      id = inactiveAccountId,
      status = AccountStatus.Closed
    )
    
    unsafeRun(repository.save(inactiveAccount))
    
    // Retrieve active accounts
    val activeAccounts = unsafeRun(repository.findActive())
    
    activeAccounts.map(_.id) should contain(accountId)
    activeAccounts.map(_.id) should not contain inactiveAccountId
  }
  
  test("update should modify an existing account") {
    val updatedAccount = testAccount.copy(
      name = "Updated Account Name",
      balance = Money(200, Currency.USD),
      updatedAt = Instant.now
    )
    
    val result = unsafeRun(repository.save(updatedAccount))
    result shouldBe updatedAccount
    
    val retrieved = unsafeRun(repository.findById(accountId)).get
    retrieved.name shouldBe "Updated Account Name"
    retrieved.balance shouldBe Money(200, Currency.USD)
  }
  
  test("delete should remove an account") {
    unsafeRun(repository.delete(accountId))
    
    val retrieved = unsafeRun(repository.findById(accountId))
    retrieved shouldBe None
  }
  
  test("save should handle duplicate IDs properly") {
    // Create account with a fixed ID
    val fixedId = AccountId(UUID.fromString("11111111-1111-1111-1111-111111111111"))
    val account1 = testAccount.copy(id = fixedId)
    
    // First save should succeed
    unsafeRun(repository.save(account1))
    
    // Create second account with same ID but different data
    val account2 = account1.copy(name = "Duplicate ID Account")
    
    // Second save should update, not insert
    val result = unsafeRun(repository.save(account2))
    result.name shouldBe "Duplicate ID Account"
    
    // Verify only one record exists
    val allAccounts = unsafeRun(repository.findByCustomer(customerId))
    allAccounts.count(_.id == fixedId) shouldBe 1
  }
```

## Common Pitfalls

1. **N+1 Query Problem**:
   - Don't query related entities in loops
   - Use joins or batch loading

2. **Eager Loading Everything**:
   - Don't always load all related entities
   - Consider lazy loading or projections for large entities

3. **Raw SQL Injection**:
   - Don't concatenate strings for SQL queries
   - Use prepared statements or query builders

4. **Missing Transaction Boundaries**:
   - Don't forget to handle transactions
   - Consider using higher-level transaction management

5. **Swallowing Exceptions**:
   - Don't catch and ignore exceptions
   - Map to appropriate domain errors with context

6. **Entity Mapping Issues**:
   - Don't lose domain invariants during mapping
   - Be careful with null values from database

7. **Missing Indexes**:
   - Don't forget to create indexes for common queries
   - Monitor query performance

8. **Connection Leaks**:
   - Don't forget to close connections, statements, and result sets
   - Use resource-safe constructs

9. **Hard-coded Queries**:
   - Don't duplicate SQL across multiple repositories
   - Consider query builders or centralized SQL definitions

## Checklist

When implementing a repository, ensure:

- [ ] Implements the corresponding repository interface from the domain layer
- [ ] Named according to technology and entity (e.g., `PostgreSQLUserRepository`)
- [ ] Database schema properly represents the entity structure
- [ ] All interface methods are implemented completely
- [ ] Proper mapping between domain entities and database records
- [ ] Error handling for common database issues (constraints, connections, etc.)
- [ ] SQL injection prevention using prepared statements
- [ ] Performance considerations for common operations
- [ ] Batch operations for bulk updates when needed
- [ ] Connection management to prevent leaks
- [ ] Transaction handling for operations that need it
- [ ] Integration tests with actual database
- [ ] No domain logic mixed with data access logic