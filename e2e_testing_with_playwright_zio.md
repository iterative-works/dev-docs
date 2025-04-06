---
status: draft
last updated: 2025-04-06
version: "0.1"
---
> [!draft] Draft Document
> This document is an initial draft and may change significantly.
# E2E Testing with Playwright and ZIO

## Overview

Our end-to-end testing approach combines Playwright for browser automation with ZIO for effect management. This integration allows us to write readable, reliable tests that verify our application's behavior from a user's perspective.

The framework is designed to be extracted into a shared library for use across multiple projects.

## Framework Components

### 1. Project Structure

We organize our e2e tests in a dedicated module:

```
ynab-importer/
  └── e2e-tests/
      ├── src/
      │   ├── test/
      │   │   ├── resources/
      │   │   │   └── application.conf  # Test configuration
      │   │   └── scala/works/iterative/incubator/e2e/
      │   │       ├── PlaywrightSupport.scala  # Core support trait
      │   │       ├── setup/
      │   │       │   └── InstallBrowsers.scala  # Browser installation
      │   │       └── tests/
      │   │           └── FeatureNameSpec.scala  # Feature tests
      │   └── main/
      │       └── scala/  # Shared code if needed
      └── build.sbt  # Module-specific build settings
```

### 2. Key Components

#### PlaywrightSupport Trait

This core trait provides ZIO wrappers around Playwright APIs:

```scala
trait PlaywrightSupport {
  // Configuration
  case class PlaywrightConfig(
    baseUrl: String,
    headless: Boolean,
    slowMo: Int,
    timeout: Int,
    // Other configuration...
  )
  
  // ZIO layers
  val playwrightLayer: ZLayer[Any, Throwable, Playwright]
  val browserLayer: ZLayer[PlaywrightConfig & Playwright, Throwable, Browser]
  val pageLayer: ZLayer[Browser & PlaywrightConfig, Throwable, Page]
  
  // Helper methods
  def navigateTo(path: String): ZIO[Page & PlaywrightConfig, Throwable, Unit]
  def click(selector: String): ZIO[Page, Throwable, Unit]
  def fill(selector: String, value: String): ZIO[Page, Throwable, Unit]
  def findElement(selector: String): ZIO[Page, Throwable, ElementHandle]
  def waitForUrlContains(urlSubstring: String): ZIO[Page, Throwable, Unit]
  // Many other helpers...
}
```

#### Test Specs

Each feature test extends ZIOSpecDefault and mixes in PlaywrightSupport:

```scala
object FeatureNameSpec extends ZIOSpecDefault with PlaywrightSupport {
  override def spec = suite("Feature Name")(
    test("Scenario description") {
      withPlaywright(
        for {
          // Test steps using PlaywrightSupport methods
          _ <- navigateTo("/path")
          _ <- click("#some-button")
          value <- getText("#result")
        } yield assertTrue(value == "expected value")
      )
    }
  )
}
```

### 3. Configuration

Tests can be configured through environment variables or application.conf:

```hocon
e2e-tests {
  baseUrl = "http://localhost:8080"
  baseUrl = ${?E2E_BASE_URL}
  
  playwright {
    headless = true
    headless = ${?E2E_HEADLESS}
    browserType = "chromium"
    # Other settings...
  }
}
```

## Best Practices

### Test Organization

1. **Feature-based structure**: Organize tests by features, matching Gherkin feature files
2. **Scenario alignment**: Each test should correspond to a scenario in the feature file
3. **Shared fixtures**: Use test fixtures for common setups to reduce duplication

### Robust Selectors

1. Use stable selectors that won't break with minor UI changes:
   - Prefer IDs and data attributes (`data-testid="..."`) over CSS classes
   - Use content where appropriate (`text=Submit`)
   - Avoid complex CSS selectors that depend on specific DOM structure

2. Wait properly for elements and navigation:
   ```scala
   // Wait for URL after navigation
   _ <- waitForUrlContains("/expected-path")
   
   // Wait for an element to appear
   _ <- waitForSelector("#element-id")
   ```

### Test Data Management

1. Create fresh test data for each test when possible
2. Clean up test data after tests complete
3. Use unique identifiers in test data to avoid conflicts

### Error Handling

1. Capture screenshots on test failures:
   ```scala
   def captureScreenshot(name: String): ZIO[Page, Throwable, Unit] = 
     ZIO.serviceWithZIO[Page] { page =>
       ZIO.attempt {
         page.screenshot(new Page.ScreenshotOptions()
           .setPath(Paths.get(s"screenshots/$name.png")))
       }
     }
   ```

2. Add detailed logging for test steps:
   ```scala
   for {
     _ <- ZIO.logInfo("Clicking login button")
     _ <- click("#login-button")
   } yield ()
   ```

## Extraction Plan

This testing framework is designed to be extracted into a shared library. The extraction will involve:

1. Moving the PlaywrightSupport trait and utilities to a shared module
2. Adding comprehensive documentation
3. Creating configurable options for different testing needs
4. Providing example test implementations
5. Adding CI integration examples

The shared library will provide a consistent approach to e2e testing across all Iterative Works projects.