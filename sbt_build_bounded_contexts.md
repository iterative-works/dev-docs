---
status: tested
last updated: 2025-04-06
version: "1.0"
---
> [!tested] Tested Document
> This document has been validated in real projects.
# SBT Build Structure for Bounded Contexts

This document outlines our approach to structuring SBT builds to support our bounded context architecture with vertical slices.

## Bounded Context Modules with Complete Vertical Slices

We structure our SBT build to align with our package organization, where each bounded context contains its complete vertical slice (domain, application, infrastructure, and web layers).

## Module Structure

```scala
// Core shared module (minimal cross-context elements)
lazy val core = (project in file("core"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "core",
    commonSettings,
    // Only essential shared dependencies
    IWMaterialsDeps.useZIO()
  )

// Complete bounded context modules with integration tests as submodules
lazy val accounts = (project in file("bounded-contexts/accounts"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "accounts",
    commonSettings,
    // All dependencies required for this bounded context
    IWMaterialsDeps.useZIO(),
    IWMaterialsDeps.quill,
    IWMaterialsDeps.tapirZIO,
    IWMaterialsDeps.tapirZIOJson
  )
  .dependsOn(core)

// Integration tests as a submodule of accounts
lazy val accountsIt = (project in file("bounded-contexts/accounts/it"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "accounts-it",
    testSettings,
    IWMaterialsDeps.useZIOTest()
  )
  .dependsOn(accounts)

lazy val transactions = (project in file("bounded-contexts/transactions"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "transactions",
    commonSettings,
    // All dependencies required for this bounded context
    IWMaterialsDeps.useZIO(),
    IWMaterialsDeps.quill,
    IWMaterialsDeps.tapirZIO,
    IWMaterialsDeps.tapirZIOJson
  )
  .dependsOn(core)

// Integration tests as a submodule of transactions
lazy val transactionsIt = (project in file("bounded-contexts/transactions/it"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "transactions-it",
    testSettings,
    IWMaterialsDeps.useZIOTest()
  )
  .dependsOn(transactions)

// Application assembly
lazy val app = (project in file("app"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "app",
    commonSettings
  )
  .dependsOn(accounts, transactions)

// End-to-end tests at top level
lazy val e2eTests = (project in file("e2e-tests"))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    name := "e2e-tests",
    testSettings
  )
  .dependsOn(app)
```

## Directory Structure

The overall directory structure looks like:

```
project-root/
├── core/                       # Core shared module
│   └── src/
│       ├── main/
│       └── test/
│
├── bounded-contexts/
│   ├── accounts/               # Accounts bounded context
│   │   ├── src/                # Main sources
│   │   │   ├── main/
│   │   │   └── test/           # Unit tests
│   │   │
│   │   └── it/                 # Integration tests submodule
│   │       └── src/
│   │           └── test/
│   │
│   └── transactions/           # Transactions bounded context
│       ├── src/                # Main sources
│       │   ├── main/
│       │   └── test/           # Unit tests
│       │
│       └── it/                 # Integration tests submodule
│           └── src/
│               └── test/
│
├── app/                        # Application assembly
│   └── src/
│       ├── main/
│       └── test/
│
└── e2e-tests/                  # End-to-end tests
    └── src/
        └── test/
```

## Module and Package Alignment

Each SBT module contains the full vertical slice for a bounded context, matching our package structure:

```
bounded-contexts/
  └── accounts/
      └── src/main/scala/works/iterative/accounts/
          ├── domain/         # Domain models
          ├── application/    # Application services
          ├── infrastructure/ # Infrastructure implementations
          └── web/           # Web components

  └── transactions/
      └── src/main/scala/works/iterative/transactions/
          ├── domain/
          ├── application/
          ├── infrastructure/
          └── web/
```

This ensures that our SBT module boundaries align perfectly with our bounded context boundaries.

## Core Module

The core module contains only truly shared elements that cross bounded contexts:

- Common value types (Money, Identifiers, etc.)
- Shared interfaces
- Cross-cutting domain concepts
- Base traits and utilities

We keep this module minimal to avoid creating hidden dependencies between bounded contexts.

## Application Module

The application module:
- Composes bounded contexts
- Provides main entry point
- Configures shared infrastructure
- Handles cross-cutting assembly concerns

## Testing Structure

Our testing approach follows three layers:

1. **Unit Tests** - Within each bounded context module alongside the code they test
   - Test individual components in isolation
   - Focused on domain logic and service behaviors
   - Use mocks or test doubles for dependencies

2. **Integration Tests** - As submodules of each bounded context
   - Test interactions with infrastructure
   - Verify repository implementations
   - Test service integrations within the bounded context
   - May use TestContainers for database testing

3. **End-to-End Tests** - At the top level
   - Test the complete application
   - Verify cross-bounded-context interactions
   - Use browser automation for UI testing
   - May use a complete integration environment

This approach allows us to maintain bounded context isolation while still testing at all required levels.

## Common Settings

We define common settings to ensure consistency across modules:

```scala
lazy val commonSettings = Seq(
  scalaVersion := "3.6.3",
  organization := "works.iterative",
  // Other common settings
)

lazy val testSettings = Seq(
  commonSettings,
  IWMaterialsDeps.useZIOTest()
)
```

## Cross-Context Communication

For communication between bounded contexts:

1. **Event-based communication** is preferred
2. **Direct dependencies** should be minimized and explicitly managed
3. **API interfaces** can be defined in the core module when necessary

When a bounded context needs to reference another:

```scala
// In app module's build.sbt
lazy val accounts = (project in file("bounded-contexts/accounts"))
  // ...

lazy val transactions = (project in file("bounded-contexts/transactions"))
  // Include dependency on accounts if necessary
  .dependsOn(accounts % "compile->compile;test->test")
  // ...
```

This explicit dependency makes it clear when bounded contexts are not fully isolated.

## Infrastructure Independence

This structure allows each bounded context to:

1. **Choose its own infrastructure** implementations
2. **Evolve independently** of other contexts
3. **Be tested in isolation**
4. **Potentially be deployed separately** if needed

## Benefits of This Approach

1. **Aligned Structure**: SBT modules match our package organization
2. **Clear Boundaries**: Each bounded context is clearly defined
3. **Complete Context**: Each module contains everything needed for the context
4. **Cohesive Testing**: Integration tests live within their bounded context
5. **Independent Evolution**: Bounded contexts can change without affecting others
6. **Easier Navigation**: All related code is in the same module hierarchy
7. **Team Ownership**: Teams can own complete bounded contexts including their tests

## Practical Considerations

When using this structure, keep in mind:

1. **Core Module Discipline**: Keep the core module minimal to avoid hidden coupling
2. **Dependency Management**: Be explicit about any cross-context dependencies
3. **Potential Duplication**: Some infrastructure code may be duplicated across contexts
4. **Package Consistency**: Maintain consistent package structure across all bounded contexts
5. **Version Alignment**: Keep shared library versions consistent across modules

This structure reinforces our architectural principles by aligning our build organization with our logical organization. It makes bounded contexts the primary unit of development, testing, and deployment.