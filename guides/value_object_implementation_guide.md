---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
---
> [!robot] AI-Generated Content
> This is AI-generated content pending human review. Information may be inaccurate or misaligned with actual processes.

# Implementation Guide: Value Object

## Purpose and Responsibilities

A Value Object is a fundamental building block of domain-driven design that:

- Represents a concept from the domain that is defined entirely by its attributes
- Has no identity beyond the combination of its attributes
- Is immutable - once created, it never changes
- Is equality-comparable based on its attributes, not by reference
- Encapsulates domain validation rules for its attributes
- Can contain business logic related to its attributes

Value Objects should:
- Be small, focused representations of domain attributes with meaning
- Enforce its invariants through validation in construction
- Provide operations that produce new instances rather than mutating state

Value Objects should NOT:
- Have identity (use Entity for domain objects with identity)
- Change state after creation
- Represent entities or aggregates
- Depend on repositories or infrastructure concerns

## Structure and Patterns

### Basic Structure

```scala
// Example structure
case class Money(amount: BigDecimal, currency: Currency):
  require(amount >= 0, "Amount must be non-negative")

  def add(other: Money): Money =
    require(currency == other.currency, "Cannot add different currencies")
    Money(amount + other.amount, currency)

  def subtract(other: Money): Money =
    require(currency == other.currency, "Cannot subtract different currencies")
    require(amount >= other.amount, "Result cannot be negative")
    Money(amount - other.amount, currency)

  def multiply(factor: BigDecimal): Money =
    Money(amount * factor, currency)

// Companion object for factory methods
object Money:
  def zero(currency: Currency): Money = Money(0, currency)

  def fromString(s: String, currency: Currency): Either[String, Money] =
    try
      val amount = BigDecimal(s)
      Right(Money(amount, currency))
    catch
      case e: NumberFormatException => Left(s"Invalid amount format: $s")
```

### Key Structural Elements

1. **Case Class Declaration**:
   - Use case classes for automatic equality and hashcode
   - Make all fields immutable (`val`, not `var`)
   - Make all fields private if they need custom accessors

2. **Validation**:
   - Use `require` for simple validations in the constructor
   - Use factory methods for complex validations
   - Return `Either[Error, ValueObject]` for validation that may fail

3. **Operations**:
   - Methods that operate on the value should return new instances
   - Make operations type-safe (e.g., can't add money of different currencies)
   - Consider using operator overloading where appropriate (`+`, `*`, etc.)

4. **Factory Methods**:
   - Place in companion object
   - Provide methods for common creation patterns
   - Consider creating DSL-like methods for fluent instantiation

## Implementation Steps

1. **Identify the Value Concept**:
   - Determine if the concept is defined purely by its attributes
   - Check if equality should be based on all attributes, not identity

2. **Define Attributes**:
   - Identify the minimal set of attributes needed to define the value
   - Use other value objects as attributes where appropriate

3. **Create Case Class**:
   - Define as a case class with immutable attributes
   - Place in the domain package of its bounded context

4. **Add Validations**:
   - Add `require` statements to constructor for validations
   - Create factory methods for complex validations

5. **Implement Operations**:
   - Add methods for domain operations that produce new instances
   - Ensure type-safety in all operations

6. **Create Companion Object**:
   - Add factory methods in companion object
   - Consider adding parsing methods if needed

7. **Write Tests**:
   - Test validation rules
   - Test operations
   - Test equality

## Examples

### Simple Example: EmailAddress

```scala
case class EmailAddress private (value: String):
  // No additional validation here since it's handled in the factory method

  def domain: String = value.split("@").last

object EmailAddress:
  private val EmailRegex = """^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$""".r

  def fromString(email: String): Either[String, EmailAddress] =
    email match
      case EmailRegex() => Right(new EmailAddress(email.toLowerCase))
      case _ => Left(s"Invalid email format: $email")
```

### Complex Example: DateRange

```scala
import java.time.LocalDate

case class DateRange private (start: LocalDate, end: LocalDate):
  require(start.isBefore(end) || start.isEqual(end), "Start date must be before or equal to end date")

  def contains(date: LocalDate): Boolean =
    (date.isEqual(start) || date.isAfter(start)) &&
    (date.isEqual(end) || date.isBefore(end))

  def overlaps(other: DateRange): Boolean =
    !(end.isBefore(other.start) || start.isAfter(other.end))

  def length: Int = java.time.Period.between(start, end).getDays

  def expandToInclude(date: LocalDate): DateRange =
    if contains(date) then this
    else if date.isBefore(start) then DateRange.between(date, end)
    else DateRange.between(start, date)

object DateRange:
  def between(start: LocalDate, end: LocalDate): Either[String, DateRange] =
    if start.isAfter(end) then
      Left("Start date must be before or equal to end date")
    else
      Right(new DateRange(start, end))

  def singleDay(date: LocalDate): DateRange = new DateRange(date, date)

  def fromNowToEndOfMonth: DateRange =
    val now = LocalDate.now
    val endOfMonth = now.withDayOfMonth(now.lengthOfMonth)
    new DateRange(now, endOfMonth)
```

## Integration Points

Value Objects typically integrate with:

1. **Entities**:
   - As attributes of entity classes
   - To enforce consistent validation of domain attributes

2. **Other Value Objects**:
   - As components of larger value objects
   - In operations between value objects (like adding two Money objects)

3. **Domain Services**:
   - As parameters and return values of domain services
   - For domain operations on values

4. **Application Services**:
   - To validate and normalize input from the outside world
   - To return well-formed domain values to clients

5. **Repositories**:
   - To store as part of entities
   - To use as query parameters

## Testing Approach

Value Objects are straightforward to test since they're pure and have no dependencies. Focus on:

1. **Validation Tests**:
   - Test constructor or factory methods with valid inputs
   - Test constructor or factory methods with invalid inputs
   - Test edge cases specific to the domain rules

2. **Operation Tests**:
   - Test each operation method with various inputs
   - Test combinations of operations
   - Test operations that should fail and verify they fail appropriately

3. **Equality Tests**:
   - Test that two value objects with same attributes are equal
   - Test that two value objects with different attributes are not equal

### Example Test:

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers

class MoneySpec extends AnyFunSuite with Matchers:
  // Validation tests
  test("Money should be created with positive amount") {
    Money(100, Currency.USD) shouldBe a[Money]
  }

  test("Money should be created with zero amount") {
    Money(0, Currency.USD) shouldBe a[Money]
  }

  test("Money should reject negative amounts") {
    an[IllegalArgumentException] should be thrownBy Money(-1, Currency.USD)
  }

  // Operation tests
  test("Adding money of same currency") {
    val m1 = Money(100, Currency.USD)
    val m2 = Money(50, Currency.USD)
    m1.add(m2) shouldBe Money(150, Currency.USD)
  }

  test("Adding money of different currency should fail") {
    val m1 = Money(100, Currency.USD)
    val m2 = Money(50, Currency.EUR)
    an[IllegalArgumentException] should be thrownBy m1.add(m2)
  }

  // Equality tests
  test("Money with same amount and currency should be equal") {
    Money(100, Currency.USD) shouldBe Money(100, Currency.USD)
  }

  test("Money with different amount should not be equal") {
    Money(100, Currency.USD) should not be Money(101, Currency.USD)
  }

  test("Money with different currency should not be equal") {
    Money(100, Currency.USD) should not be Money(100, Currency.EUR)
  }
```

## Common Pitfalls

1. **Mutability**:
   - Don't create value objects with mutable state
   - Ensure all methods return new instances rather than modifying state

2. **Missing Validations**:
   - Don't skip validation of invariants
   - Validate in constructors or factory methods, not in usage sites

3. **Leaking Implementation Details**:
   - Don't expose internal representation if it doesn't match the conceptual model
   - Consider private constructors with factory methods for complex validations

4. **Equality Issues**:
   - Don't use reference equality for value objects
   - Be careful with floating point equality comparisons

5. **Primitive Obsession**:
   - Don't use primitive types (String, Int) directly when a value object better captures domain concepts
   - Example: Use `EmailAddress` instead of `String`, `Money` instead of `Double`

6. **Missing Domain Logic**:
   - Don't forget to add domain operations on value objects
   - Value objects should be more than data carriers; they should implement domain behaviors

## Checklist

When implementing a value object, ensure:

- [ ] The concept is truly defined by its attributes, not identity
- [ ] All attributes are immutable (`val`, not `var`)
- [ ] All required validations are implemented
- [ ] All domain operations return new instances
- [ ] Proper equality implementation (correctly inherited from case class)
- [ ] Factory methods for complex object creation
- [ ] Comprehensive tests covering validation, operations, and equality
- [ ] Documentation explaining the domain concept
- [ ] No dependencies on repositories or other infrastructure concerns
- [ ] Follows naming conventions (noun or adjective phrase)
