---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
---
> [!robot] AI-Generated Content
> This is AI-generated content pending human review. Information may be inaccurate or misaligned with actual processes.

# Implementation Guide: Entity

## Purpose and Responsibilities

An Entity is a core domain object that:

- Has a distinct identity that persists across state changes
- Maintains continuity and traceability throughout its lifecycle
- Contains behavior related to its domain role
- Encapsulates business rules and invariants
- Manages its own internal state transitions

Entities should:
- Have a clear identity concept (typically an ID)
- Protect their invariants when state changes
- Implement domain behaviors related to their responsibilities
- Use value objects for their attributes when appropriate

Entities should NOT:
- Be defined solely by their attributes (use Value Objects instead)
- Directly access infrastructure concerns like databases
- Contain business logic that spans multiple entities (use Domain Services)

## Structure and Patterns

### Basic Structure

```scala
// Basic entity structure
case class User(
  id: UserId,                      // Identity attribute
  email: EmailAddress,             // Value object attribute
  name: String,
  role: UserRole,                  // Enumeration or value object
  status: UserStatus,              // Current state
  createdAt: Instant,              // Lifecycle attribute
  updatedAt: Instant               // Lifecycle attribute
):
  // Domain behavior
  def activate: User =
    if status == UserStatus.Pending then
      this.copy(
        status = UserStatus.Active,
        updatedAt = Instant.now
      )
    else
      this

  def changeEmail(newEmail: EmailAddress): User =
    this.copy(
      email = newEmail,
      updatedAt = Instant.now
    )

  def hasPermission(permission: Permission): Boolean =
    role.hasPermission(permission)

// Companion object for factory methods
object User:
  def create(
    id: UserId,
    email: EmailAddress,
    name: String,
    role: UserRole
  ): User =
    val now = Instant.now
    User(
      id = id,
      email = email,
      name = name,
      role = role,
      status = UserStatus.Pending,
      createdAt = now,
      updatedAt = now
    )
```

### Key Structural Elements

1. **Case Class Declaration**:
   - Use case classes for automatic equality, hashcode, and copy methods
   - Use immutable fields for all properties
   - Make the ID the primary constructor parameter

2. **Identity Attribute**:
   - Use a typed ID (preferably a value object) rather than primitive types
   - Make the ID immutable after creation

3. **State Attributes**:
   - Use value objects rather than primitives where appropriate
   - Include lifecycle attributes (creation date, update date)

4. **Behavior Methods**:
   - Implement methods that model domain behavior
   - Methods that change state should return a new entity instance
   - Validate state transitions to maintain invariants

5. **Factory Methods**:
   - Place creation logic in companion object
   - Use factory methods to enforce valid initial state

## Implementation Steps

1. **Identify the Entity Concept**:
   - Determine if the concept has identity independent of its attributes
   - Check if the concept has a lifecycle that needs to be tracked

2. **Define Identity**:
   - Create a strongly typed ID (usually a value object)
   - Decide on ID generation strategy (client-generated, system-generated)

3. **Define Attributes**:
   - Identify state attributes needed for the entity
   - Use value objects for complex attributes
   - Include lifecycle attributes (creation date, modification date)

4. **Create Case Class**:
   - Define as a case class with immutable attributes
   - Place in the domain package of its bounded context

5. **Implement Behavior**:
   - Add methods for domain operations
   - Ensure state changes respect entity invariants
   - Make state-changing operations return new instances

6. **Create Companion Object**:
   - Add factory methods in companion object
   - Implement validation logic for entity creation

7. **Define Repository Interface**:
   - Create repository interface to define persistence operations
   - Place in the same domain package as the entity

8. **Write Tests**:
   - Test state transitions and behavior
   - Test invariant enforcement
   - Test factory methods

## Examples

### Simple Example: Account

```scala
import java.time.Instant

case class AccountId(value: UUID)

case class Account(
  id: AccountId,
  name: String,
  balance: Money,
  status: AccountStatus,
  createdAt: Instant,
  updatedAt: Instant
):
  def deposit(amount: Money): Either[String, Account] =
    if amount.amount <= 0 then
      Left("Deposit amount must be positive")
    else if status != AccountStatus.Active then
      Left(s"Cannot deposit to account with status $status")
    else
      Right(
        this.copy(
          balance = balance.add(amount),
          updatedAt = Instant.now
        )
      )

  def withdraw(amount: Money): Either[String, Account] =
    if amount.amount <= 0 then
      Left("Withdrawal amount must be positive")
    else if status != AccountStatus.Active then
      Left(s"Cannot withdraw from account with status $status")
    else if balance.amount < amount.amount then
      Left("Insufficient funds")
    else
      Right(
        this.copy(
          balance = balance.subtract(amount),
          updatedAt = Instant.now
        )
      )

  def close: Either[String, Account] =
    if balance.amount > 0 then
      Left("Cannot close account with positive balance")
    else
      Right(
        this.copy(
          status = AccountStatus.Closed,
          updatedAt = Instant.now
        )
      )

object Account:
  def create(
    id: AccountId,
    name: String,
    initialBalance: Money,
    status: AccountStatus = AccountStatus.Active
  ): Account =
    val now = Instant.now
    Account(
      id = id,
      name = name,
      balance = initialBalance,
      status = status,
      createdAt = now,
      updatedAt = now
    )
```

### Complex Example: Order

```scala
import java.time.Instant

case class OrderId(value: UUID)

case class Order(
  id: OrderId,
  customerId: CustomerId,
  items: List[OrderItem],
  status: OrderStatus,
  shippingAddress: Address,
  paymentMethod: PaymentMethod,
  total: Money,
  createdAt: Instant,
  updatedAt: Instant
):
  require(items.nonEmpty, "Order must have at least one item")

  def addItem(item: OrderItem): Either[String, Order] =
    status match
      case OrderStatus.Draft =>
        val newItems = items :+ item
        val newTotal = calculateTotal(newItems)
        Right(
          this.copy(
            items = newItems,
            total = newTotal,
            updatedAt = Instant.now
          )
        )
      case _ =>
        Left(s"Cannot add items when order is in $status status")

  def removeItem(itemId: OrderItemId): Either[String, Order] =
    status match
      case OrderStatus.Draft =>
        val newItems = items.filterNot(_.id == itemId)
        if newItems.isEmpty then
          Left("Order must have at least one item")
        else
          val newTotal = calculateTotal(newItems)
          Right(
            this.copy(
              items = newItems,
              total = newTotal,
              updatedAt = Instant.now
            )
          )
      case _ =>
        Left(s"Cannot remove items when order is in $status status")

  def submit: Either[String, Order] =
    status match
      case OrderStatus.Draft =>
        Right(
          this.copy(
            status = OrderStatus.Submitted,
            updatedAt = Instant.now
          )
        )
      case _ =>
        Left(s"Cannot submit order in $status status")

  def cancel: Either[String, Order] =
    status match
      case OrderStatus.Draft | OrderStatus.Submitted =>
        Right(
          this.copy(
            status = OrderStatus.Cancelled,
            updatedAt = Instant.now
          )
        )
      case OrderStatus.Shipped | OrderStatus.Delivered =>
        Left("Cannot cancel order that has been shipped or delivered")
      case _ =>
        Left(s"Cannot cancel order in $status status")

  private def calculateTotal(orderItems: List[OrderItem]): Money =
    orderItems.foldLeft(Money.zero(Currency.USD)) { (acc, item) =>
      acc.add(item.price.multiply(item.quantity))
    }

object Order:
  def create(
    id: OrderId,
    customerId: CustomerId,
    items: List[OrderItem],
    shippingAddress: Address,
    paymentMethod: PaymentMethod
  ): Either[String, Order] =
    if items.isEmpty then
      Left("Order must have at least one item")
    else
      val total = items.foldLeft(Money.zero(Currency.USD)) { (acc, item) =>
        acc.add(item.price.multiply(item.quantity))
      }

      val now = Instant.now
      Right(
        Order(
          id = id,
          customerId = customerId,
          items = items,
          status = OrderStatus.Draft,
          shippingAddress = shippingAddress,
          paymentMethod = paymentMethod,
          total = total,
          createdAt = now,
          updatedAt = now
        )
      )
```

## Integration Points

Entities typically integrate with:

1. **Value Objects**:
   - As attributes of the entity
   - To encapsulate domain concepts and validation

2. **Other Entities**:
   - Through relationships (containment, references by ID)
   - In domain operations spanning multiple entities

3. **Domain Services**:
   - When operations involve multiple entities or external logic

4. **Repository Interfaces**:
   - To define how the entity is persisted and retrieved
   - To manage the entity's lifecycle

5. **Application Services**:
   - To orchestrate entity operations in use cases
   - To map between DTOs and entities

## Testing Approach

Entities should be tested thoroughly since they contain core domain behavior:

1. **Creation Tests**:
   - Test factory methods with valid inputs
   - Test factory methods with invalid inputs
   - Verify initial state is correct

2. **State Transition Tests**:
   - Test each method that changes entity state
   - Verify state transitions maintain invariants
   - Test illegal state transitions

3. **Behavior Tests**:
   - Test domain behavior methods with various inputs
   - Test combinations of operations
   - Verify business rules are enforced

### Example Test:

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers

class AccountSpec extends AnyFunSuite with Matchers:
  val accountId = AccountId(UUID.randomUUID)
  val initialBalance = Money(100, Currency.USD)

  // Creation tests
  test("Account should be created with initial values") {
    val account = Account.create(accountId, "Savings Account", initialBalance)

    account.id shouldBe accountId
    account.name shouldBe "Savings Account"
    account.balance shouldBe initialBalance
    account.status shouldBe AccountStatus.Active
  }

  // Behavior tests
  test("Deposit should increase the balance") {
    val account = Account.create(accountId, "Savings Account", initialBalance)
    val depositAmount = Money(50, Currency.USD)

    val updatedAccount = account.deposit(depositAmount).getOrElse(fail)

    updatedAccount.balance shouldBe Money(150, Currency.USD)
    updatedAccount.updatedAt should be > account.updatedAt
  }

  test("Withdraw should decrease the balance") {
    val account = Account.create(accountId, "Savings Account", initialBalance)
    val withdrawAmount = Money(50, Currency.USD)

    val updatedAccount = account.withdraw(withdrawAmount).getOrElse(fail)

    updatedAccount.balance shouldBe Money(50, Currency.USD)
    updatedAccount.updatedAt should be > account.updatedAt
  }

  test("Withdraw should fail if insufficient funds") {
    val account = Account.create(accountId, "Savings Account", initialBalance)
    val withdrawAmount = Money(150, Currency.USD)

    val result = account.withdraw(withdrawAmount)

    result.isLeft shouldBe true
    result.left.getOrElse("") should include("Insufficient funds")
  }

  test("Close should set status to Closed") {
    val account = Account.create(accountId, "Savings Account", Money(0, Currency.USD))

    val closedAccount = account.close.getOrElse(fail)

    closedAccount.status shouldBe AccountStatus.Closed
  }

  test("Close should fail if balance is positive") {
    val account = Account.create(accountId, "Savings Account", initialBalance)

    val result = account.close

    result.isLeft shouldBe true
    result.left.getOrElse("") should include("positive balance")
  }

  // Invalid operations tests
  test("Cannot deposit to closed account") {
    val account = Account.create(
      accountId,
      "Savings Account",
      Money(0, Currency.USD),
      AccountStatus.Closed
    )
    val depositAmount = Money(50, Currency.USD)

    val result = account.deposit(depositAmount)

    result.isLeft shouldBe true
    result.left.getOrElse("") should include("status")
  }
```

## Common Pitfalls

1. **Anemic Models**:
   - Don't create entities with only getters and setters
   - Include domain behavior and business rules

2. **Missing Identity**:
   - Don't use primitive types for IDs
   - Create strongly typed IDs (usually value objects)

3. **Exposing Internal State**:
   - Don't provide setters that bypass business rules
   - Use behavior methods that enforce invariants

4. **Infrastructure Concerns**:
   - Don't include database access or external services in entities
   - Keep entities free of infrastructure dependencies

5. **Overusing Entities**:
   - Don't use entities for concepts defined by attributes (use Value Objects)
   - Don't include behavior that spans multiple entities (use Domain Services)

6. **Mutable State**:
   - Don't use mutable fields in a functional approach
   - Return new instances for state changes

7. **Complex Constructors**:
   - Don't expose complex construction logic in constructors
   - Use factory methods in companion objects

## Checklist

When implementing an entity, ensure:

- [ ] The concept truly has identity independent of its attributes
- [ ] Identity is represented with a strongly typed ID
- [ ] All attributes are immutable
- [ ] Value objects are used for complex attributes
- [ ] Domain behavior methods enforce invariants
- [ ] State changes return new instances
- [ ] Factory methods enforce valid initial state
- [ ] Repository interface is defined for persistence
- [ ] Comprehensive tests covering creation, state transitions, and behavior
- [ ] No infrastructure dependencies
- [ ] Follows naming conventions (noun or noun phrase)
