---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
---

# Implementation Guide: Application Service

## Purpose and Responsibilities

An Application Service is a component that:

- Orchestrates domain objects to fulfill specific use cases
- Acts as a facade between the domain layer and the outside world
- Handles cross-cutting concerns like transactions and security
- Translates between domain objects and DTOs/ViewModels
- Manages the workflow of business operations

Application Services should:
- Coordinate domain objects without containing domain logic
- Express use cases in terms of domain operations
- Handle transactional boundaries
- Validate inputs before passing to domain objects
- Return well-defined result types

Application Services should NOT:
- Contain domain logic (use Domain Services and Entities)
- Access UI or presentation logic
- Be stateful
- Bypass domain objects to manipulate data directly
- Contain complex conditional logic (use domain objects)

## Structure and Patterns

### Basic Structure

```scala
// Basic application service
class TransactionImportService(
  // Dependencies expressed as interfaces defined in the domain
  transactionRepository: TransactionRepository,
  sourceAccountRepository: SourceAccountRepository,
  transactionMatcher: TransactionMatcher,  // Domain service
  transactionCategorizer: TransactionCategorizer,  // Domain service
  clock: Clock,  // System dependency
  idGenerator: IdGenerator  // System dependency
):
  /**
   * Import transactions from a source account.
   * 
   * @param command The import command with account ID and transactions
   * @return An import result with statistics and status
   */
  def importTransactions(
    command: ImportTransactionsCommand
  ): ZIO[Any, ApplicationError, ImportTransactionsResult] =
    for
      // Step 1: Validate and load required data
      sourceAccount <- sourceAccountRepository.findById(command.sourceAccountId)
        .someOrFail(EntityNotFoundError(s"Source account not found: ${command.sourceAccountId}"))
      
      // Step 2: Transform input data to domain objects
      rawTransactions = command.rawTransactions.map(raw => 
        Transaction(
          id = idGenerator.nextId(),
          sourceAccountId = sourceAccount.id,
          date = raw.date,
          amount = raw.amount,
          description = raw.description,
          status = TransactionStatus.Pending,
          createdAt = clock.instant(),
          updatedAt = clock.instant()
        )
      )
      
      // Step 3: Use domain service to match transactions
      existingTransactions <- transactionRepository.findBySourceAccount(sourceAccount.id)
      matchingResult = transactionMatcher.matchTransactions(
        rawTransactions, 
        existingTransactions
      )
      
      // Step 4: Use domain service to categorize unmatched transactions
      categorizedTransactions = transactionCategorizer.categorizeTransactions(
        matchingResult.unmatchedTransactions
      )
      
      // Step 5: Save the new transactions
      savedTransactions <- ZIO.foreach(categorizedTransactions)(tx => 
        transactionRepository.save(tx)
      )
      
      // Step 6: Create and return a result object
      importResult = ImportTransactionsResult(
        totalImported = savedTransactions.size,
        duplicatesSkipped = matchingResult.matches.size,
        categorizedCount = categorizedTransactions.count(_.category.isDefined),
        importedTransactions = savedTransactions,
        status = if savedTransactions.isEmpty && matchingResult.matches.nonEmpty then
          ImportStatus.AllDuplicates
        else if savedTransactions.nonEmpty then
          ImportStatus.Success
        else
          ImportStatus.NoTransactions
      )
    yield importResult
```

### Key Structural Elements

1. **Class Declaration**:
   - Define as a regular class (not a trait)
   - Name according to use case (`TransactionImportService` for importing transactions)
   - Accept all dependencies as constructor parameters

2. **Dependencies**:
   - Depend on abstractions/interfaces from the domain layer
   - Express external dependencies through interfaces
   - Avoid direct dependencies on infrastructure classes

3. **Method Signatures**:
   - Name methods after use cases or operations
   - Accept command/query objects as parameters
   - Return result objects wrapped in appropriate effect types
   - Use explicit error types

4. **Implementation Flow**:
   - Validate inputs early
   - Load required domain objects
   - Delegate to domain services/entities for logic
   - Handle result translation
   - Manage transactions

5. **Error Handling**:
   - Define application-specific error types
   - Handle domain errors and map to application errors
   - Use ZIO error channel for failures

## Implementation Steps

1. **Identify Use Cases**:
   - Define distinct operations that your application needs to support
   - Group related operations into services

2. **Define Commands/Queries and Results**:
   - Create input DTOs (commands/queries) for operations
   - Create result DTOs to return

3. **Identify Required Dependencies**:
   - Determine which repositories, domain services, and external services are needed
   - Express dependencies as interfaces from the domain layer

4. **Implement Service Methods**:
   - Create methods for each use case
   - Implement orchestration flow
   - Keep methods focused on a single use case

5. **Add Validation Logic**:
   - Validate inputs before processing
   - Use validators or validate inline

6. **Test the Service**:
   - Write tests with mocked dependencies
   - Test different scenarios and error cases

## Examples

### Simple Example: UserRegistrationService

```scala
class UserRegistrationService(
  userRepository: UserRepository,
  passwordHasher: PasswordHasher,
  emailService: EmailService,
  idGenerator: IdGenerator,
  clock: Clock
):
  /**
   * Register a new user.
   * 
   * @param command The registration command with user details
   * @return The created user or registration errors
   */
  def registerUser(
    command: RegisterUserCommand
  ): ZIO[Any, ApplicationError, RegistrationResult] =
    for
      // Validate unique email
      existingUser <- userRepository.findByEmail(EmailAddress.fromString(command.email).toOption.get)
      _ <- ZIO.when(existingUser.isDefined)(ZIO.fail(EmailAlreadyExistsError(command.email)))
      
      // Create user entity
      user = User(
        id = UserId(idGenerator.nextId()),
        email = EmailAddress.fromString(command.email).toOption.get,
        name = command.name,
        hashedPassword = passwordHasher.hash(command.password),
        role = UserRole.Regular,
        status = UserStatus.Pending,
        createdAt = clock.instant(),
        updatedAt = clock.instant()
      )
      
      // Save user
      savedUser <- userRepository.save(user)
      
      // Send welcome email (fire and forget)
      _ <- emailService.sendWelcomeEmail(user.email).fork
    yield RegistrationResult(
      userId = savedUser.id.toString,
      registrationDate = savedUser.createdAt,
      status = savedUser.status
    )

// Command and Result DTOs
case class RegisterUserCommand(
  email: String,
  name: String,
  password: String
)

case class RegistrationResult(
  userId: String,
  registrationDate: Instant,
  status: UserStatus
)

// Error types
sealed trait ApplicationError extends Throwable
case class EmailAlreadyExistsError(email: String) extends ApplicationError
case class EntityNotFoundError(message: String) extends ApplicationError
```

### Complex Example: OrderProcessingService

```scala
class OrderProcessingService(
  orderRepository: OrderRepository,
  customerRepository: CustomerRepository,
  productRepository: ProductRepository,
  inventoryService: InventoryService,
  paymentProcessor: PaymentProcessor, // Domain service
  shippingCalculator: ShippingCalculator, // Domain service
  emailService: EmailService,
  idGenerator: IdGenerator,
  clock: Clock
):
  /**
   * Create a new order.
   * 
   * @param command Order creation command with order details
   * @return The created order or errors
   */
  def createOrder(
    command: CreateOrderCommand
  ): ZIO[Any, ApplicationError, OrderCreationResult] = 
    for
      // Get required entities
      customer <- customerRepository.findById(command.customerId)
        .someOrFail(EntityNotFoundError(s"Customer not found: ${command.customerId}"))
      
      productIds = command.items.map(_.productId).toSet
      products <- ZIO.foreach(productIds)(id => 
        productRepository.findById(id)
          .someOrFail(EntityNotFoundError(s"Product not found: $id"))
      ).map(_.map(p => p.id -> p).toMap)
      
      // Check inventory
      inventoryCheck <- ZIO.foreach(command.items)(item => 
        inventoryService.checkAvailability(item.productId, item.quantity)
          .mapError(e => InventoryError(s"Inventory check failed: ${e.getMessage}"))
      )
      
      unavailableItems = inventoryCheck.zip(command.items).filter(!_._1.available)
      _ <- ZIO.when(unavailableItems.nonEmpty)(ZIO.fail(
        InventoryError(s"Some items are not available: ${unavailableItems.map(_._2.productId).mkString(", ")}")
      ))
      
      // Create order items
      orderItems = command.items.map(item => {
        val product = products(item.productId)
        OrderItem(
          id = OrderItemId(idGenerator.nextId()),
          productId = product.id,
          name = product.name,
          price = product.price,
          quantity = item.quantity
        )
      })
      
      // Calculate shipping
      shippingAddress = Address.fromAddressDTO(command.shippingAddress)
      shippingCost = shippingCalculator.calculateShipping(
        items = orderItems,
        destination = shippingAddress,
        shippingMethod = command.shippingMethod
      )
      
      // Create order entity
      orderId = OrderId(idGenerator.nextId())
      now = clock.instant()
      order <- ZIO.fromEither(Order.create(
        id = orderId,
        customerId = customer.id,
        items = orderItems,
        shippingAddress = shippingAddress,
        paymentMethod = command.paymentMethod
      )).mapError(msg => ValidationError(msg))
      
      // Process payment
      paymentResult = paymentProcessor.processPayment(
        order = order,
        paymentMethod = command.paymentMethod
      )
      
      // Handle payment result
      finalOrder <- paymentResult match
        case PaymentResult.Approved(transactionId) =>
          val withPayment = order.copy(
            status = OrderStatus.Submitted,
            paymentTransactionId = Some(transactionId),
            updatedAt = clock.instant()
          )
          orderRepository.save(withPayment)
            .tap(_ => inventoryService.reserveItems(orderId, orderItems))
            .tap(_ => emailService.sendOrderConfirmation(customer.email, withPayment))
            
        case PaymentResult.Rejected(reason) =>
          val failedOrder = order.copy(
            status = OrderStatus.PaymentFailed,
            paymentFailureReason = Some(reason),
            updatedAt = clock.instant()
          )
          orderRepository.save(failedOrder)
            .tap(_ => emailService.sendPaymentFailedNotification(customer.email, failedOrder))
    yield OrderCreationResult(
      orderId = finalOrder.id.toString,
      status = finalOrder.status,
      total = finalOrder.total,
      createdAt = finalOrder.createdAt,
      paymentSuccessful = finalOrder.status == OrderStatus.Submitted
    )

  /**
   * Cancel an existing order.
   * 
   * @param command Order cancellation command
   * @return Cancellation result or errors
   */
  def cancelOrder(
    command: CancelOrderCommand
  ): ZIO[Any, ApplicationError, OrderCancellationResult] =
    for
      // Get order
      orderId = OrderId.fromString(command.orderId)
        .getOrElse(return ZIO.fail(ValidationError(s"Invalid order ID: ${command.orderId}")))
      
      order <- orderRepository.findById(orderId)
        .someOrFail(EntityNotFoundError(s"Order not found: $orderId"))
      
      // Apply domain cancel operation
      cancelledOrder <- ZIO.fromEither(order.cancel)
        .mapError {
          case msg if msg.contains("shipped") => 
            BusinessRuleViolationError("Cannot cancel an order that has been shipped")
          case msg => 
            BusinessRuleViolationError(msg)
        }
      
      // Update order and perform side effects
      savedOrder <- orderRepository.save(cancelledOrder)
      
      // Release inventory if needed
      _ <- ZIO.when(order.status == OrderStatus.Submitted) {
        inventoryService.releaseItems(orderId, order.items)
      }
      
      // Notify customer
      customer <- customerRepository.findById(order.customerId)
        .someOrFail(EntityNotFoundError(s"Customer not found: ${order.customerId}"))
      
      _ <- emailService.sendOrderCancellationConfirmation(customer.email, savedOrder)
    yield OrderCancellationResult(
      orderId = savedOrder.id.toString,
      status = savedOrder.status,
      cancelledAt = savedOrder.updatedAt
    )

// Commands and results
case class CreateOrderCommand(
  customerId: String,
  items: Seq[OrderItemCommand],
  shippingAddress: AddressDTO,
  shippingMethod: ShippingMethod,
  paymentMethod: PaymentMethod
)

case class OrderItemCommand(
  productId: String,
  quantity: Int
)

case class AddressDTO(
  street: String,
  city: String,
  state: String,
  zipCode: String,
  country: String
)

case class OrderCreationResult(
  orderId: String,
  status: OrderStatus,
  total: Money,
  createdAt: Instant,
  paymentSuccessful: Boolean
)

case class CancelOrderCommand(
  orderId: String,
  reason: Option[String]
)

case class OrderCancellationResult(
  orderId: String,
  status: OrderStatus,
  cancelledAt: Instant
)

// Additional error types
case class ValidationError(message: String) extends ApplicationError
case class InventoryError(message: String) extends ApplicationError
case class PaymentError(message: String) extends ApplicationError
case class BusinessRuleViolationError(message: String) extends ApplicationError
```

## Integration Points

Application Services typically integrate with:

1. **Repositories**:
   - Load and persist domain entities
   - Manage transactional boundaries

2. **Domain Services**:
   - Delegate domain logic operations
   - Use for operations spanning multiple entities

3. **Domain Entities**:
   - Create and manipulate entities
   - Invoke entity behavior methods

4. **External Services**:
   - Interact with external systems via ports/interfaces
   - Convert between domain models and external formats

5. **Web Controllers/Modules**:
   - Receive commands/queries from UI layer
   - Return results to be presented to users

6. **Command/Query Objects**:
   - Accept structured input data
   - Validate before processing

7. **Result Objects**:
   - Return structured result data
   - Map domain objects to client-friendly formats

## Testing Approach

Application Services should be tested thoroughly since they coordinate use cases:

1. **Unit Testing with Mocks**:
   - Mock all dependencies
   - Test different scenarios by controlling mock behavior
   - Verify correct dependency interactions

2. **Integration Testing**:
   - Test with real repositories (using test databases)
   - Verify end-to-end flow through service

3. **Error Case Testing**:
   - Test all error paths
   - Verify appropriate error responses

### Example Test:

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers
import zio.{Runtime, Unsafe, ZIO}

class UserRegistrationServiceSpec extends AnyFunSuite with Matchers:
  // Test dependencies
  val userRepository = mock[UserRepository]
  val passwordHasher = mock[PasswordHasher]
  val emailService = mock[EmailService]
  val idGenerator = mock[IdGenerator]
  val clock = mock[Clock]
  
  // System under test
  val service = UserRegistrationService(
    userRepository, 
    passwordHasher, 
    emailService, 
    idGenerator, 
    clock
  )
  
  // Helper to run ZIO effects in tests
  def unsafeRun[E, A](zio: ZIO[Any, E, A]): A =
    Unsafe.unsafe { implicit unsafe =>
      Runtime.default.unsafe.run(zio).getOrThrowFiberFailure()
    }
  
  test("Register user successfully when email is unique") {
    // Setup mocks
    val email = "test@example.com"
    val emailAddress = EmailAddress.fromString(email).toOption.get
    val password = "password123"
    val hashedPassword = "hashed_password"
    val userId = UUID.randomUUID()
    val now = Instant.now
    
    when(userRepository.findByEmail(emailAddress))
      .thenReturn(ZIO.succeed(None))
    
    when(passwordHasher.hash(password))
      .thenReturn(hashedPassword)
    
    when(idGenerator.nextId())
      .thenReturn(userId)
    
    when(clock.instant())
      .thenReturn(now)
    
    val newUser = User(
      id = UserId(userId),
      email = emailAddress,
      name = "Test User",
      hashedPassword = hashedPassword,
      role = UserRole.Regular,
      status = UserStatus.Pending,
      createdAt = now,
      updatedAt = now
    )
    
    when(userRepository.save(any[User]))
      .thenReturn(ZIO.succeed(newUser))
    
    when(emailService.sendWelcomeEmail(emailAddress))
      .thenReturn(ZIO.succeed(()))
    
    // Execute
    val command = RegisterUserCommand(
      email = email,
      name = "Test User",
      password = password
    )
    
    val result = unsafeRun(service.registerUser(command))
    
    // Verify
    result.userId shouldBe userId.toString
    result.status shouldBe UserStatus.Pending
    result.registrationDate shouldBe now
    
    verify(userRepository).save(any[User])
    verify(emailService).sendWelcomeEmail(emailAddress)
  }
  
  test("Registration fails when email already exists") {
    // Setup mocks
    val email = "existing@example.com"
    val emailAddress = EmailAddress.fromString(email).toOption.get
    val existingUser = User(
      id = UserId(UUID.randomUUID()),
      email = emailAddress,
      name = "Existing User",
      hashedPassword = "hashedpw",
      role = UserRole.Regular,
      status = UserStatus.Active,
      createdAt = Instant.now(),
      updatedAt = Instant.now()
    )
    
    when(userRepository.findByEmail(emailAddress))
      .thenReturn(ZIO.succeed(Some(existingUser)))
    
    // Execute
    val command = RegisterUserCommand(
      email = email,
      name = "Test User",
      password = "password123"
    )
    
    val exception = intercept[EmailAlreadyExistsError] {
      unsafeRun(service.registerUser(command))
    }
    
    // Verify
    exception.email shouldBe email
    verify(userRepository, never()).save(any[User])
    verify(emailService, never()).sendWelcomeEmail(any[EmailAddress])
  }
```

## Common Pitfalls

1. **Domain Logic in Application Services**:
   - Don't put domain logic in application services
   - Delegate to domain entities and services

2. **Transaction Management**:
   - Don't forget transactional boundaries
   - Ensure operations are atomic when needed

3. **Inadequate Error Handling**:
   - Don't use generic exceptions
   - Define specific error types
   - Handle all error cases

4. **Direct Infrastructure Dependencies**:
   - Don't depend directly on infrastructure implementations
   - Use interfaces from the domain layer

5. **Excessive Logic in Services**:
   - Don't create bloated service methods
   - Break down complex operations into smaller methods

6. **Validation Issues**:
   - Don't forget to validate inputs early
   - Don't duplicate validation logic across services

7. **Leaking Domain Objects**:
   - Don't return domain entities directly to UI layer
   - Map to specific result DTOs

8. **Missing Side Effects**:
   - Don't forget important side effects (notifications, etc.)
   - Consider making non-critical side effects fire-and-forget

## Checklist

When implementing an application service, ensure:

- [ ] Service is named according to the use case it implements
- [ ] All dependencies are injected and defined as interfaces
- [ ] Commands and results are clearly defined
- [ ] Input validation occurs early in the flow
- [ ] Domain logic is delegated to domain objects
- [ ] Error handling is comprehensive
- [ ] Transactional boundaries are properly defined
- [ ] Domain objects are mapped to appropriate result types
- [ ] Complex operations are broken down into manageable steps
- [ ] Tests cover successful paths and error scenarios
- [ ] Documentation explains the use case and flow