---
status: ai-generated
last updated: {{DATE}}
version: "0.1"
---

# Component Implementation Guide Priorities

This document outlines the prioritized order for creating component implementation guides, based on our architecture and development approach.

## First Priority Components

These components form the core of our domain model and are essential for implementing any feature following our Functional Core approach:

1. **Value Object**
   - **Justification**: Basic building blocks of our domain model that encapsulate business rules for attributes
   - **Usage Frequency**: Very High - Used in almost every feature
   - **Complexity**: Low to Medium - Crucial to get right but conceptually straightforward
   - **Sample Components**: `Money`, `DateRange`, `EmailAddress`, `TransactionId`

2. **Entity**
   - **Justification**: Core domain objects that maintain identity and lifecycle
   - **Usage Frequency**: High - Central to most domain models
   - **Complexity**: Medium - Requires clear understanding of identity vs attributes
   - **Sample Components**: `Account`, `Transaction`, `User`

3. **Repository Interface**
   - **Justification**: Critical boundary between domain and infrastructure, defined by domain needs
   - **Usage Frequency**: High - Nearly every entity needs persistence
   - **Complexity**: Medium - Must express domain intent without implementation details
   - **Sample Components**: `TransactionRepository`, `AccountRepository`

4. **Domain Service**
   - **Justification**: Encapsulates domain operations that don't belong to a single entity
   - **Usage Frequency**: Medium-High - Common for cross-entity operations
   - **Complexity**: Medium - Requires clear domain understanding
   - **Sample Components**: `TransactionMatcher`, `FeeCalculator`

5. **Application Service**
   - **Justification**: Orchestrates use cases and composes domain components
   - **Usage Frequency**: High - Central to implementing use cases
   - **Complexity**: Medium-High - Requires understanding of component composition
   - **Sample Components**: `TransactionImportService`, `UserRegistrationService`

## Second Priority Components

These components extend the core model with implementation details and specialized behaviors:

6. **Repository Implementation**
   - **Justification**: Connects domain interfaces to actual data storage
   - **Usage Frequency**: High - Needed for every repository interface
   - **Complexity**: Medium-High - Requires infrastructure knowledge
   - **Sample Components**: `PostgreSQLTransactionRepository`

7. **Web Module**
   - **Justification**: Organizes UI components and routes for a feature
   - **Usage Frequency**: Medium-High - One per feature area
   - **Complexity**: Medium - Requires understanding of ZIO HTTP and ScalaTags
   - **Sample Components**: `TransactionImportModule`, `UserManagementModule`

8. **Domain Event**
   - **Justification**: Enables event-driven communication between domains
   - **Usage Frequency**: Medium - Important for complex domains
   - **Complexity**: Medium - Requires understanding of event patterns
   - **Sample Components**: `TransactionImported`, `AccountCreated`

9. **Command/Query**
   - **Justification**: Structures input for application services
   - **Usage Frequency**: Medium-High - Used with application services
   - **Complexity**: Low - Mostly data structures
   - **Sample Components**: `ImportTransactionsCommand`, `GetAccountBalanceQuery`

10. **Result**
    - **Justification**: Structures output from application services
    - **Usage Frequency**: Medium-High - Used with application services
    - **Complexity**: Low - Mostly data structures
    - **Sample Components**: `ImportSummaryResult`, `AccountInfoResult`

## Third Priority Components

These components handle more specialized concerns or are used less frequently:

11. **Port**
    - **Justification**: Defines interfaces to external systems
    - **Usage Frequency**: Medium - Used for external integrations
    - **Complexity**: Medium - Requires clear boundaries
    - **Sample Components**: `EmailSender`, `PaymentGateway`

12. **Adapter**
    - **Justification**: Implements ports to connect to external systems
    - **Usage Frequency**: Medium - One per port
    - **Complexity**: Medium-High - Requires external system knowledge
    - **Sample Components**: `SMTPEmailSender`, `StripePaymentGateway`

13. **View**
    - **Justification**: Renders UI elements
    - **Usage Frequency**: Medium-High - Multiple per web module
    - **Complexity**: Medium - Requires UI design knowledge
    - **Sample Components**: `TransactionListView`, `UserProfileView`

14. **Aggregate**
    - **Justification**: Manages cluster of related entities
    - **Usage Frequency**: Low-Medium - Used for complex domain relationships
    - **Complexity**: High - Requires deep domain understanding
    - **Sample Components**: `Order` (containing `OrderLines`, `ShippingInfo`)

15. **Effect Definition**
    - **Justification**: Abstracts required effects in functional core
    - **Usage Frequency**: Medium - Used for external system effects
    - **Complexity**: Medium-High - Requires ZIO effect system understanding
    - **Sample Components**: `BankingEffect`, `NotificationEffect`

## Implementation Plan

Based on this prioritization, we will develop implementation guides in the following phases:

### Phase 1: Core Domain Components
- Value Object Implementation Guide
- Entity Implementation Guide
- Domain Service Implementation Guide
- Repository Interface Implementation Guide
- Application Service Implementation Guide

### Phase 2: Infrastructure and UI Components
- Repository Implementation Guide
- Web Module Implementation Guide
- Domain Event Implementation Guide
- Command/Query Implementation Guide
- Result Implementation Guide

### Phase 3: Specialized Components
- Port Implementation Guide
- Adapter Implementation Guide
- View Implementation Guide
- Aggregate Implementation Guide
- Effect Definition Implementation Guide

## Component Guide Template

Each implementation guide will follow this structure:

```
# Implementation Guide: [Component Type]

## Purpose and Responsibilities
- What this component type is for
- What it should and shouldn't do

## Structure and Patterns
- Required fields/methods
- Standard patterns
- Naming conventions

## Implementation Steps
1. Step 1: [Description]
2. Step 2: [Description]
3. ...

## Examples
- Simple example with explanation
- Complex example with explanation

## Integration Points
- How this component typically interacts with others
- Common composition patterns

## Testing Approach
- How to test this component type
- Example test cases

## Common Pitfalls
- What to avoid
- Frequent issues

## Checklist
- [ ] Component follows single responsibility principle
- [ ] All required methods implemented
- [ ] Tests cover all behaviors
- [ ] Documentation added
```
