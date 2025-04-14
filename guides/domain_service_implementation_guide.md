---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
---

# Implementation Guide: Domain Service

## Purpose and Responsibilities

A Domain Service is a stateless component that:

- Encapsulates domain logic that doesn't naturally belong to a single entity or value object
- Orchestrates operations between multiple domain objects
- Implements business processes and domain rules
- Maintains the purity of the functional core
- Expresses domain concepts that are verb-like rather than noun-like

Domain Services should:
- Implement operations involving multiple domain objects
- Express domain concepts as behaviors
- Be stateless and side-effect free (pure)
- Use domain language in their naming and API
- Return new instances rather than modifying state

Domain Services should NOT:
- Hold state between operations
- Access infrastructure concerns directly (repositories, external systems)
- Implement CRUD operations (use repositories)
- Replace entity behavior that naturally belongs to entities
- Include application-specific logic or orchestration (use Application Services)

## Structure and Patterns

### Basic Structure

```scala
// Basic domain service
trait TransactionMatcher:
  /**
   * Matches imported transactions with existing ones based on
   * transaction properties to prevent duplicates.
   *
   * @param newTransactions Newly imported transactions to match
   * @param existingTransactions Existing transactions to match against
   * @return A MatchingResult containing matched and unmatched transactions
   */
  def matchTransactions(
    newTransactions: Seq[Transaction],
    existingTransactions: Seq[Transaction]
  ): MatchingResult

// Service implementation
case class TransactionMatcherImpl() extends TransactionMatcher:
  override def matchTransactions(
    newTransactions: Seq[Transaction],
    existingTransactions: Seq[Transaction]
  ): MatchingResult =
    val matches = for
      newTx <- newTransactions
      existingTx <- existingTransactions
      if areMatching(newTx, existingTx)
    yield (newTx, existingTx)
    
    val matchedNew = matches.map(_._1).toSet
    val unmatched = newTransactions.filterNot(matchedNew.contains)
    
    MatchingResult(
      matches = matches.toMap,
      unmatchedTransactions = unmatched
    )
  
  private def areMatching(tx1: Transaction, tx2: Transaction): Boolean =
    // Implementation of matching logic
    tx1.date == tx2.date &&
    tx1.amount == tx2.amount &&
    tx1.description == tx2.description

// Result type
case class MatchingResult(
  matches: Map[Transaction, Transaction],
  unmatchedTransactions: Seq[Transaction]
)
```

### Key Structural Elements

1. **Service Interface**:
   - Define as a trait
   - Name as a verb or process (`TransactionMatcher`, not `TransactionService`)
   - Place in the domain package

2. **Method Signatures**:
   - Use domain types for parameters and returns
   - Define clear, descriptive method names
   - Express domain operations and intent

3. **Implementation**:
   - Implement as case class (typically)
   - Make it pure (no side effects)
   - Extract complex operations to private methods

4. **Result Types**:
   - Create specific result types to encapsulate complex operation results
   - Use algebraic data types (case classes, sealed traits) for results

## Implementation Steps

1. **Identify Domain Operations**:
   - Identify logic that doesn't fit naturally in entities or value objects
   - Recognize operations that involve multiple domain objects

2. **Define Service Interface**:
   - Create a trait with domain operation methods
   - Name it according to its responsibility
   - Use domain language in method names

3. **Create Result Types**:
   - Define types to represent operation outcomes
   - Ensure they capture all relevant information

4. **Implement Service Logic**:
   - Write pure implementations of service methods
   - Extract complex operations to private helper methods
   - Ensure all operations return new instances rather than modifying state

5. **Test Service**:
   - Write unit tests with various inputs
   - Verify behavior matches domain expectations

## Examples

### Simple Example: PaymentProcessor

```scala
// Payment processing domain service
trait PaymentProcessor:
  /**
   * Process a payment for an order.
   * 
   * @param order The order to process payment for
   * @param paymentMethod The payment method to use
   * @return The result of the payment processing
   */
  def processPayment(
    order: Order,
    paymentMethod: PaymentMethod
  ): PaymentResult

case class PaymentProcessorImpl() extends PaymentProcessor:
  override def processPayment(
    order: Order,
    paymentMethod: PaymentMethod
  ): PaymentResult =
    // Domain logic for payment processing
    paymentMethod match
      case CreditCard(number, _, _) if isBlacklisted(number) =>
        PaymentResult.Rejected("Card is blacklisted")
        
      case CreditCard(_, _, cvv) if !isValidCvv(cvv) =>
        PaymentResult.Rejected("Invalid CVV")
        
      case BankTransfer(accountNumber) if !isValidBankAccount(accountNumber) =>
        PaymentResult.Rejected("Invalid bank account")
        
      case _ if order.total.amount <= 0 =>
        PaymentResult.Rejected("Invalid order amount")
        
      case _ =>
        // In a real implementation, this would return different results
        // based on more complex validation rules
        PaymentResult.Approved("PAY-" + UUID.randomUUID().toString)
  
  private def isBlacklisted(cardNumber: String): Boolean =
    // Domain logic to check if a card is blacklisted
    cardNumber.startsWith("1111")
  
  private def isValidCvv(cvv: String): Boolean =
    // Domain logic to validate CVV
    cvv.length == 3 || cvv.length == 4
    
  private def isValidBankAccount(accountNumber: String): Boolean =
    // Domain logic to validate bank account
    accountNumber.length >= 8

sealed trait PaymentResult
object PaymentResult:
  case class Approved(transactionId: String) extends PaymentResult
  case class Rejected(reason: String) extends PaymentResult
```

### Complex Example: LoanApplicationEvaluator

```scala
// Complex domain service for loan application evaluation
trait LoanApplicationEvaluator:
  /**
   * Evaluates a loan application based on various criteria
   * and determines if it should be approved.
   * 
   * @param application The loan application to evaluate
   * @param applicantHistory The applicant's credit history
   * @param currentLoans The applicant's existing loans
   * @return Detailed evaluation result
   */
  def evaluate(
    application: LoanApplication,
    applicantHistory: CreditHistory,
    currentLoans: Seq[Loan]
  ): EvaluationResult

case class LoanApplicationEvaluatorImpl(
  riskCalculator: RiskCalculator,
  loanLimits: LoanLimits
) extends LoanApplicationEvaluator:
  override def evaluate(
    application: LoanApplication,
    applicantHistory: CreditHistory,
    currentLoans: Seq[Loan]
  ): EvaluationResult =
    // Step 1: Calculate debt-to-income ratio
    val monthlyIncome = application.income.monthly
    val existingDebt = currentLoans.map(_.payment.monthly).sum
    val proposedPayment = calculateMonthlyPayment(
      application.amount, 
      application.term, 
      application.interestRate
    )
    val debtToIncomeRatio = (existingDebt + proposedPayment) / monthlyIncome
    
    // Step 2: Check credit score against thresholds
    val creditScore = applicantHistory.score
    val hasDelinquencies = applicantHistory.delinquencies > 0
    
    // Step 3: Calculate risk score
    val riskScore = riskCalculator.calculateRisk(
      creditScore = creditScore,
      debtToIncomeRatio = debtToIncomeRatio,
      loanAmount = application.amount,
      hasDelinquencies = hasDelinquencies,
      employmentYears = application.employmentYears
    )
    
    // Step 4: Apply loan policy rules
    val evaluations = Seq(
      evaluateCreditScore(creditScore),
      evaluateDebtToIncome(debtToIncomeRatio),
      evaluateLoanAmount(application.amount),
      evaluateEmployment(application.employmentYears),
      evaluateDelinquencies(applicantHistory.delinquencies)
    )
    
    // Step 5: Make decision based on all factors
    val approved = 
      riskScore <= loanLimits.maxRiskScore &&
      !evaluations.exists(_.factor == EvaluationFactor.Critical) &&
      evaluations.count(_.result == EvaluationOutcome.Fail) <= 1
    
    val maxApprovedAmount = if approved then
      calculateMaxAmount(
        application.income, 
        existingDebt, 
        application.term, 
        application.interestRate
      )
    else
      Money.zero(application.amount.currency)
    
    EvaluationResult(
      approved = approved,
      riskScore = riskScore,
      factorEvaluations = evaluations,
      maxApprovedAmount = maxApprovedAmount,
      suggestedInterestRate = 
        if approved then calculateAdjustedRate(application.interestRate, riskScore)
        else BigDecimal(0)
    )
  
  private def calculateMonthlyPayment(
    principal: Money,
    termMonths: Int,
    annualRate: BigDecimal
  ): Money =
    // Complex loan payment calculation
    val monthlyRate = annualRate / 12 / 100
    val payment = principal.amount * monthlyRate * math.pow(1 + monthlyRate, termMonths) / 
                 (math.pow(1 + monthlyRate, termMonths) - 1)
    Money(BigDecimal(payment), principal.currency)
  
  private def evaluateCreditScore(score: Int): FactorEvaluation =
    if score >= loanLimits.minCreditScore then
      FactorEvaluation(EvaluationFactor.CreditScore, EvaluationOutcome.Pass)
    else if score >= loanLimits.minCreditScore - 50 then
      FactorEvaluation(EvaluationFactor.CreditScore, EvaluationOutcome.Marginal)
    else
      FactorEvaluation(EvaluationFactor.CreditScore, EvaluationOutcome.Fail, 
                       Some("Credit score too low"))
  
  // Additional evaluation methods omitted for brevity
  
  private def calculateMaxAmount(
    income: Income,
    existingDebt: Money,
    term: Int,
    rate: BigDecimal
  ): Money =
    // Calculate max loan amount based on income and existing debt
    // (Complex calculation omitted for brevity)
    Money(income.monthly.amount * 36, income.monthly.currency)
    
  private def calculateAdjustedRate(
    baseRate: BigDecimal,
    riskScore: Int
  ): BigDecimal =
    // Adjust interest rate based on risk score
    baseRate + (riskScore / 100.0)

// Supporting types
case class EvaluationResult(
  approved: Boolean,
  riskScore: Int,
  factorEvaluations: Seq[FactorEvaluation],
  maxApprovedAmount: Money,
  suggestedInterestRate: BigDecimal
)

case class FactorEvaluation(
  factor: EvaluationFactor,
  result: EvaluationOutcome,
  reason: Option[String] = None
)

enum EvaluationFactor:
  case CreditScore, DebtToIncome, LoanAmount, Employment, 
       PaymentHistory, Critical

enum EvaluationOutcome:
  case Pass, Marginal, Fail

trait RiskCalculator:
  def calculateRisk(
    creditScore: Int,
    debtToIncomeRatio: BigDecimal,
    loanAmount: Money,
    hasDelinquencies: Boolean,
    employmentYears: Int
  ): Int

case class LoanLimits(
  maxLoanAmount: Money,
  minCreditScore: Int,
  maxDebtToIncomeRatio: BigDecimal,
  maxRiskScore: Int
)
```

## Integration Points

Domain Services typically integrate with:

1. **Domain Entities**:
   - Operate on entities as input and output
   - May create or modify entities (returning new instances)

2. **Value Objects**:
   - Use value objects for parameters and calculations
   - May create new value objects as part of operations

3. **Other Domain Services**:
   - Compose with other services for more complex operations
   - Delegate specialized operations to other services

4. **Application Services**:
   - Called by application services as part of use cases
   - Provide domain logic to application orchestration

5. **Tests**:
   - Easy to test due to pure functions
   - No external dependencies required for testing

## Testing Approach

Domain Services are typically pure functions, making them easy to test:

1. **Unit Testing**:
   - Test with a variety of input combinations
   - Verify outputs match expected business rules
   - Test edge cases and boundary conditions

2. **Property-Based Testing**:
   - Define properties that should hold true for any valid input
   - Use property testing frameworks to generate test cases

3. **BDD-Style Testing**:
   - Test real business scenarios described in domain language
   - Focus on behavior rather than implementation

### Example Test:

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers

class PaymentProcessorSpec extends AnyFunSuite with Matchers:
  val processor = PaymentProcessorImpl()
  val validOrder = Order(
    id = OrderId(UUID.randomUUID),
    // other fields omitted for brevity
    total = Money(100, Currency.USD)
  )
  
  // Acceptance tests for payment processing
  test("Valid credit card payments are approved") {
    val paymentMethod = CreditCard("4111111111111111", "12/25", "123")
    
    val result = processor.processPayment(validOrder, paymentMethod)
    
    result.isInstanceOf[PaymentResult.Approved] shouldBe true
  }
  
  test("Blacklisted credit cards are rejected") {
    val paymentMethod = CreditCard("1111111111111111", "12/25", "123")
    
    val result = processor.processPayment(validOrder, paymentMethod)
    
    result match
      case PaymentResult.Rejected(reason) => 
        reason should include("blacklisted")
      case _ => fail("Expected payment to be rejected")
  }
  
  test("Zero amount orders are rejected") {
    val zeroAmountOrder = validOrder.copy(
      total = Money(0, Currency.USD)
    )
    val paymentMethod = CreditCard("4111111111111111", "12/25", "123")
    
    val result = processor.processPayment(zeroAmountOrder, paymentMethod)
    
    result match
      case PaymentResult.Rejected(reason) => 
        reason should include("amount")
      case _ => fail("Expected payment to be rejected")
  }
  
  // More tests would follow for other cases...
```

## Common Pitfalls

1. **Stateful Services**:
   - Don't hold state between operations
   - Services should be purely functional

2. **Entity Behavior in Services**:
   - Don't move entity behavior to services
   - Only include operations that span multiple entities

3. **Infrastructure Dependencies**:
   - Don't directly depend on repositories or external systems
   - Express needs abstractly via interfaces

4. **Application Logic in Domain Services**:
   - Don't include use-case specific orchestration
   - Focus on domain logic independent of application flow

5. **Generic "Service" Names**:
   - Don't use vague names like "AccountService"
   - Name services after the operation or process they perform

6. **Anemic Services**:
   - Don't create services that are just CRUD wrappers
   - Focus on meaningful domain operations

7. **Mutable Operations**:
   - Don't mutate input parameters
   - Return new instances instead

## Checklist

When implementing a domain service, ensure:

- [ ] The service represents a domain concept that's a behavior or process
- [ ] Named after the domain process it implements (verb-like)
- [ ] Service methods use domain types (entities, value objects)
- [ ] Operations are pure and side-effect free
- [ ] Logic spans multiple entities or value objects
- [ ] Specific result types defined for complex operations
- [ ] No direct infrastructure dependencies
- [ ] All operations return new instances rather than modifying state
- [ ] Comprehensive unit tests for all scenarios
- [ ] Documentation explains domain purpose and rules