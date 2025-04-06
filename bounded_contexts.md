---
status: in-review
last updated: 2025-04-06
version: "0.1"
---
> [!review] In-Review Document
> This document is under review and feedback is welcome.
# Bounded Contexts in Domain-Driven Design

## Definition

A Bounded Context is a central pattern in Domain-Driven Design (DDD) that defines the scope and applicability of a particular domain model. It establishes clear boundaries where a specific model is consistently valid and maintains its integrity.

## Key Characteristics

A Bounded Context is:
- A conceptual boundary where a specific domain model is consistently applicable
- The scope within which a particular ubiquitous language has clear, consistent meaning
- An area where specific business rules and constraints apply
- Often aligned with a specific business capability or function

## How to Identify Bounded Contexts

1. **Language Boundaries**: When the same term means different things in different parts of the system (e.g., "account" in banking vs. user management)

2. **Business Capability Alignment**: Distinct business functions that operate semi-independently

3. **Data Ownership**: Areas with clear responsibility for certain data entities

4. **Team Boundaries**: Different teams often work on different bounded contexts

5. **Change Patterns**: Parts of the system that tend to change together

6. **Integration Points**: Natural seams in the system where integration occurs

7. **Process Completeness**: A complete business process with clear inputs and outputs

8. **Autonomous Operation**: Can function relatively independently of other parts

## Benefits

- Clarifies domain models and their scope of application
- Prevents model bleed and concept contamination
- Allows different teams to work on different contexts
- Enables strategic design decisions about integration
- Focuses development on areas that deliver business value
- Reduces cognitive load by creating smaller, coherent models

## Relation to Other DDD Concepts

- **Ubiquitous Language**: Each bounded context has its own ubiquitous language
- **Context Mapping**: Defines how bounded contexts interact with each other
- **Aggregates**: Live within a single bounded context
- **Domain Events**: Often signal communication across bounded context boundaries