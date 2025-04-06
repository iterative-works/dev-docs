---
status: tested
last updated: 2025-04-06
version: "1.0"
---
> [!tested] Tested Document
> This document has been validated in real projects.
# SBT Build Practices

We are using Nix and direnv to load our environment, see `.envrc` and `flake.nix` in project's root for details if needed.

We are running one SBT server manually during development in the project root folder and letting Metals or IDEA connect to it remotely. We are running sbt actions using `sbtn` command, eg. `sbtn compile`.

SBT is using consistent project prefixes to run the tasks, meaning if you want to run task `compile` in project `integration-tests`, you call `sbtn integration-tests/compile`.

If you want to make sure that we compile the whole source code, use `Test` configuration to do so, because this depends on the production code, not the other way around, eg. `sbtn integration-tests/Test/compile`.

## Single build.sbt Approach

We use a single `build.sbt` file for our projects rather than distributed files in subdirectories. This approach provides several benefits:

- **Centralized configuration**: All project settings are in one place, making it easier to understand the overall project structure.
- **Simplified dependency management**: Dependencies and version management can be handled consistently.
- **Easier cross-project configurations**: References between modules are clearer when defined in a single file.
- **Better IDE integration**: Some IDEs handle a single build file better than distributed ones.

We only use distributed build files (in subdirectories) when there are notable benefits, such as:
- Very large projects with many modules where the main build file becomes unwieldy
- Specialized build requirements for specific modules that would clutter the main build file
- When modules need to be built independently of the main project

## IW plugin presets

We have created a sbt plugin that can setup the usual plugins and rules for projects we use. Using this plugin in the project enables us to keep the majority of dependencies in one place and upgrade whole sets of deps just by incrementing the plugin version in project dependencies.

To use the presets, put the following in `project/project/plugins.sbt`:

```scala
resolvers += "IW releases" at "https://dig.iterative.works/maven/releases"

resolvers += "IW snapshots" at "https://dig.iterative.works/maven/snapshots"

addSbtPlugin("works.iterative.sbt" % "sbt-iw-plugin-presets" % "0.3.25")
```
## Using IWScalaProjectPlugin

Our `IWScalaProjectPlugin` sets up standard project defaults for all our Scala projects. The plugin:

- Sets Scala version defaults (current default is Scala 3.6.3)
- Enables SemanticDB for tooling support
- Configures early-semver versioning
- Integrates with ScalafixPlugin and ScalafmtPlugin
- Sets test logging preferences

To apply these settings, it should be enough to enable the plugin in root module:

```scala
lazy val root = (project in file("."))
  .enablePlugins(IWScalaProjectPlugin)
  .settings(
    // Your module-specific settings...
  )
```

## Using IWMaterialsDeps

We leverage `IWMaterialsDeps` to standardize dependency management. This provides predefined dependency groups and individual library references with consistent versioning from `IWMaterialsVersions`.

```scala
// Common dependencies using IWMaterialsDeps
lazy val commonDependencies = IWMaterialsDeps.useZIO() ++ Seq(
    IWMaterialsDeps.zioLogging,
    IWMaterialsDeps.zioJson
)

// Example usage in a module
lazy val myModule = (project in file("my-module"))
    .settings(name := "my-module")
    .enablePlugins(IWScalaProjectPlugin)
    .settings(
        commonDependencies,
        IWMaterialsDeps.scalatags,
        IWMaterialsDeps.tapirCore
    )
```

Key dependency groups:
- `useZIO()` - Core ZIO dependencies with test framework
- `useZIOAll()` - Comprehensive ZIO stack (core, streams, config, logging, etc.)
- `useZIOJson` - ZIO JSON with proper compiler options
- `useZIOTest()` - ZIO test framework
- `useScalaJavaTimeAndLocales()` - Time and localization support for Scala.js

Benefits of using IWMaterialsDeps:
- **Consistent versions**: All dependencies are coordinated through `IWMaterialsVersions`
- **Reduced boilerplate**: Common dependency sets are pre-defined
- **Simplified updates**: One place to update versions for all projects
- **Cross-platform support**: Many dependencies support JS/JVM with `%%%`

## Plugin Support with IWPluginPresets

For project plugins, we use `IWPluginPresets` to ensure consistent plugin versions across projects:

```scala
// In project/plugins.sbt
// Add core IW projects support
addIWProjects
  
// Add Scala.js support if needed
addScalaJSSupport
```

Key plugin presets include:
- `addIWProjects` - Core IW project settings and plugins
- `addScalaJSSupport` - Complete Scala.js ecosystem (scalajs, crossproject, tzdb, locales)
- `addPlaySupport` - Complete Play framework support
- Various individual plugins like `addScalaJS`, `addTzdb`, etc.

## Publishing to IW Maven Repository

To publish to the IW Maven repository, use:

```scala
// In build.sbt
publishToIW

// Also make sure your project resolves from IW repositories
resolveIW
```

This automatically handles snapshot vs. release repositories based on the version string.

## Module Structure

We organize our projects into modules following these guidelines:

1. Root project aggregates all modules
2. Core modules contain domain models and interfaces
3. Infrastructure modules implement interfaces with concrete technologies
4. Web/UI modules provide user interfaces
5. Test modules contain tests that span multiple modules

Example module structure:
```scala
lazy val core = (project in file("core"))    
    .settings(
        // Domain models, interfaces
        IWMaterialsDeps.useZIO()
    )

lazy val infrastructure = (project in file("infrastructure"))
    .dependsOn(core)
    .settings(
        // Implementations of core interfaces
        IWMaterialsDeps.useZIO(),
        IWMaterialsDeps.quill
    )

lazy val web = (project in file("web"))
    .dependsOn(core, infrastructure)
    .settings(
        // Web UI components
        IWMaterialsDeps.tapirZIO,
        IWMaterialsDeps.tapirZIOJson
    )

lazy val root = (project in file("."))
	// Enable the plugin on the root to make sure we have basic structure set up
	.enablePlugins(IWScalaProjectPlugin)
    .aggregate(core, infrastructure, web)
    .settings(
        // Root project that ties everything together
        name := "my-project"
    )
```

## Available Library Dependencies

`IWMaterialsDeps` provides a comprehensive set of library dependencies with managed versions. Below is the reference of available dependencies grouped by category.

### How to Use

For individual libraries:
```scala
// Single dependency
.settings(IWMaterialsDeps.zioJson)

// Multiple dependencies 
.settings(
  IWMaterialsDeps.tapirCore,
  IWMaterialsDeps.tapirZIO,
  IWMaterialsDeps.tapirZIOJson
)
```

For dependency groups:
```scala
// Complete ZIO stack
IWMaterialsDeps.useZIOAll()

// Core ZIO with test framework
IWMaterialsDeps.useZIO()

// ZIO test framework only
IWMaterialsDeps.useZIOTest()

// ZIO JSON with compiler options
IWMaterialsDeps.useZIOJson
```

### ZIO Ecosystem
- `zio` - Core ZIO runtime
- `zioTest`, `zioTestSbt` - ZIO test framework
- `zioConfig`, `zioConfigTypesafe`, `zioConfigMagnolia` - ZIO configuration
- `zioJson` - ZIO JSON
- `zioLogging`, `zioLoggingSlf4j` - ZIO logging
- `zioPrelude` - Functional abstractions
- `zioStreams` - ZIO streaming
- `zioInteropCats`, `zioInteropReactiveStreams` - Interop libraries
- `zioNIO` - Non-blocking I/O
- `zioCli` - Command-line interface
- `zioSchema`, `zioSchemaDerivation` - Schema derivation

### Akka Ecosystem
- `akkaActor`, `akkaActorTyped` - Core actors
- `akkaStream` - Reactive streams
- `akkaPersistenceTyped`, `akkaPersistenceQuery` - Event sourcing
- `akkaPersistenceJdbc` - JDBC journal
- `akkaPersistenceTestkit` - Testing persistence
- `akkaHttp`, `akkaHttpSprayJson` - HTTP server and client

### Database Access
- `slick`, `slickHikaricp` - Functional-relational mapping
- `quill` - Compile-time query generation for ZIO
- `magnum`, `magnumZIO`, `magnumPG` - Compile-time SQL for Scala 3

### HTTP Clients & Servers
- `sttpClient3Core`, `sttpClient4Core` - HTTP client libraries
- `http4sBlazeServer` - HTTP4S server
- `http4sPac4J`, `pac4jOIDC` - Authentication
- `tapirCore`, `tapirZIO`, `tapirZIOJson`, etc. - Declarative HTTP endpoints

### GraphQL
- `caliban`, `calibanHttp4s`, `calibanTapir` - GraphQL for Scala

### Testing
- `scalaTest`, `scalaTestPlusScalacheck` - Testing frameworks
- `playScalaTest` - Play framework testing

### Play Framework
- `play`, `playServer`, `playAkkaServer` - Web framework
- `playMailer`, `playAhcWs` - Email and WS clients
- `playJson` - JSON library

### Frontend/UI
- `scalatags` - HTML templating
- `laminar` - Reactive UI library
- `laminextCore`, `laminextTailwind`, `laminextFetch`, etc. - Laminar extensions
- `waypoint` - Client-side routing
- `urlDsl` - URL handling

### Data Transformation
- `ducktape` - Functional object mapping
- `chimney` - Data transformation

### Internationalization
- Methods:
  - `useScalaJavaTimeAndLocales(proj: Project)` - Setup time and locale plugins
  - `addScalaJavaTime`, `addScalaJavaLocales` - Individual libraries

### Miscellaneous
- `logbackClassic` - Logging implementation
- `useElastic4S` - Elasticsearch client
- `scalaJsMacroTaskExector`, `scalaJsJavaSecureRandom` - Scala.js utilities

## Best Practices

1. Define common settings and dependencies in val blocks at the top of the build file
2. Use consistent naming conventions for modules and settings
3. Reference dependency versions from `IWVersions` or keep versions at the top of the build file
4. Use `.enablePlugins(IWScalaProjectPlugin)` for root module
5. Define explicit module dependencies with `.dependsOn()`
6. Keep the module graph clean – avoid circular dependencies
7. Use `aggregate` in the root project to ensure all modules are built
8. For Scala.js projects, use the standard crossProject pattern with `useScalaJavaTimeAndLocales`
9. Always prefer `IWDeps` over direct library dependencies when available