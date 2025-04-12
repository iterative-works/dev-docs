---
status: ai-generated
created: 2025-04-09
last_updated: 2025-04-09
version: 0.1
tags:
  - zio
  - testing
  - scala
  - functional-programming
---
> [!robot] AI-Generated Content
> This is AI-generated content pending human review.

# ZIO Test Guide

## Introduction

ZIO Test is a zero-dependency testing library that makes it easy to test effectual programs. It's designed around the concept of treating tests as first-class values that can be composed and manipulated like any other ZIO effect.

### Key Features

- **First-class Tests**: Tests are ordinary ZIO values
- **Native ZIO Integration**: Testing effectful programs is as natural as testing pure ones
- **Composable Assertions**: Assertions can be combined using logical operators
- **Test Aspects**: Cross-cutting concerns can be applied to tests using modular aspects
- **Rich Assertions Library**: Comprehensive built-in assertions for various data types

### Why ZIO Test?

Traditional testing frameworks struggle with effectful code, requiring indirect transformations and complex setups. ZIO Test eliminates these issues by:

1. Treating tests as ordinary ZIO effects
2. Providing a unified syntax for testing both pure and effectful code
3. Allowing the full power of ZIO's functionality for resource management, retries, and timeouts

## Getting Started

### Dependencies

Add the following to your build.sbt:

```scala
libraryDependencies ++= Seq(
  "dev.zio" %% "zio-test"     % "2.0.19" % "test",
  "dev.zio" %% "zio-test-sbt" % "2.0.19" % "test"
)
```

### Basic Structure

Create a test suite by extending `ZIOSpecDefault`:

```scala
import zio._
import zio.test._

object MySpec extends ZIOSpecDefault {
  def spec = suite("My Test Suite")(
    test("my first test") {
      assertTrue(1 + 1 == 2)
    }
  )
}
```

## Writing Tests

### Test Structure

The basic building blocks for ZIO Test are:

- `suite`: Groups related tests together
- `test`: Defines a single test case

```scala
import zio._
import zio.test._

object CalculatorSpec extends ZIOSpecDefault {
  def spec = suite("Calculator")(
    test("addition works") {
      assertTrue(1 + 1 == 2)
    },
    test("subtraction works") {
      assertTrue(1 - 1 == 0)
    }
  )
}
```

### Testing ZIO Effects

Testing effects is straightforward:

```scala
import zio._
import zio.test._

object EffectSpec extends ZIOSpecDefault {
  def spec = suite("Effect Tests")(
    test("successful effect") {
      for {
        result <- ZIO.succeed(42)
      } yield assertTrue(result == 42)
    },
    test("failing effect") {
      for {
        exit <- ZIO.fail("error").exit
      } yield assertTrue(exit.isFailure)
    }
  )
}
```

### Nested Suites

Tests can be organized in nested suites:

```scala
import zio._
import zio.test._

object NestedSpec extends ZIOSpecDefault {
  def spec = suite("Parent Suite")(
    suite("Child Suite 1")(
      test("test 1") {
        assertTrue(true)
      }
    ),
    suite("Child Suite 2")(
      test("test 2") {
        assertTrue(true)
      }
    )
  )
}
```

## Assertions

ZIO Test provides a rich set of assertions for verifying test conditions.

### Classic Assertions

For asserting ordinary values:

```scala
import zio.test._
import zio.test.Assertion._

test("assert examples") {
  assert(2 + 2)(equalTo(4))
}
```

For asserting ZIO effects:

```scala
import zio._
import zio.test._
import zio.test.Assertion._

test("assertZIO examples") {
  assertZIO(ZIO.succeed(2 + 2))(equalTo(4))
}
```

### Smart Assertions

A unified syntax for both values and effects using the `assertTrue` macro:

```scala
import zio._
import zio.test._

test("assertTrue examples") {
  for {
    a <- ZIO.succeed(2)
    b <- ZIO.succeed(2)
  } yield assertTrue(a + b == 4)
}
```

### Built-in Assertions

ZIO Test provides a comprehensive set of built-in assertions for different data types:

#### General Assertions

```scala
import zio.test._
import zio.test.Assertion._

test("general assertions") {
  assert(42)(
    equalTo(42) &&        // Equality
    isGreaterThan(40) &&  // Numeric comparison
    isLessThan(50)        // Numeric comparison
  )
}
```

#### Collection Assertions

```scala
import zio.test._
import zio.test.Assertion._

test("collection assertions") {
  assert(List(1, 2, 3))(
    hasSize(equalTo(3)) &&    // Check size
    contains(2) &&            // Contains element
    forall(isGreaterThan(0))  // All elements satisfy condition
  )
}
```

#### String Assertions

```scala
import zio.test._
import zio.test.Assertion._

test("string assertions") {
  assert("Hello, World!")(
    containsString("Hello") &&      // Contains substring
    startsWithString("Hello") &&    // Starts with prefix
    endsWithString("World!") &&     // Ends with suffix
    matchesRegex(".*World.*")       // Matches regex
  )
}
```

#### Option Assertions

```scala
import zio.test._
import zio.test.Assertion._

test("option assertions") {
  assert(Some(42))(
    isSome(equalTo(42))  // Is Some and contains value
  ) &&
  assert(None)(isNone)   // Is None
}
```

#### Either Assertions

```scala
import zio.test._
import zio.test.Assertion._

test("either assertions") {
  assert(Right(42))(
    isRight(equalTo(42))  // Is Right and contains value
  ) &&
  assert(Left("error"))(
    isLeft(equalTo("error"))  // Is Left and contains value
  )
}
```

#### Exit/Cause Assertions

```scala
import zio._
import zio.test._
import zio.test.Assertion._

test("exit assertions") {
  for {
    successExit <- ZIO.succeed(42).exit
    failureExit <- ZIO.fail("error").exit
  } yield {
    assert(successExit)(succeeds(equalTo(42))) &&
    assert(failureExit)(fails(equalTo("error")))
  }
}
```

## Test Aspects

Test aspects are used to modify the behavior of tests by applying cross-cutting concerns.

### Common Test Aspects

Here are some commonly used test aspects:

```scala
import zio._
import zio.test._
import zio.test.TestAspect._

test("my test") {
  assertTrue(true)
} @@ timeout(10.seconds)  // Test fails if not completed within 10 seconds
  @@ retry(Schedule.recurs(3))  // Retry the test up to 3 times
  @@ flaky  // Mark the test as flaky
  @@ ignore  // Ignore the test
```

#### Execution Control Aspects

- `timeout(duration)`: Fails the test if it doesn't complete within the specified duration
- `eventually`: Retries the test until it succeeds
- `repeats(n)`: Repeats the test n times
- `retries(n)`: Retries the test n times if it fails
- `nonFlaky`: Runs the test 100 times to ensure it's not flaky

#### Filtering Aspects

- `ignore`: Ignores the test
- `jvmOnly`: Runs the test only on the JVM
- `jsOnly`: Runs the test only on JS
- `nativeOnly`: Runs the test only on Native
- `scala2Only`: Runs the test only on Scala 2
- `scala3Only`: Runs the test only on Scala 3

#### Test Transformation Aspects

- `failing`: Inverts the test result (failing tests pass, passing tests fail)
- `flaky`: Marks the test as flaky
- `sequential`: Runs tests sequentially rather than in parallel
- `timed`: Logs the execution time of the test

#### Service Aspects

- `withLiveClock`: Uses the live Clock service
- `withLiveConsole`: Uses the live Console service
- `withLiveRandom`: Uses the live Random service
- `withLiveSystem`: Uses the live System service

### Composing Test Aspects

Test aspects can be composed using the `@@` operator:

```scala
import zio._
import zio.test._
import zio.test.TestAspect._

test("composing aspects") {
  assertTrue(true)
} @@ timeout(10.seconds) @@ retries(3) @@ jvmOnly
```

The order of test aspects matters. For example:

```scala
test("order matters") {
  // This will run on the JVM, and repeat 5 times
  assertTrue(true)
} @@ jvmOnly @@ repeat(Schedule.recurs(5))

// Different than
test("different order") {
  // This will repeat 5 times only if on the JVM
  assertTrue(true)
} @@ repeat(Schedule.recurs(5)) @@ jvmOnly
```

#### Applying Aspects to Suites

Test aspects can be applied to entire suites:

```scala
import zio._
import zio.test._
import zio.test.TestAspect._

suite("My Suite")(
  test("test 1") {
    assertTrue(true)
  },
  test("test 2") {
    assertTrue(true)
  }
) @@ timeout(30.seconds)  // Apply timeout to all tests in the suite
```

## Test Resources

ZIO Test integrates with ZIO's resource management for test fixtures:

```scala
import zio._
import zio.test._

object ResourceSpec extends ZIOSpecDefault {
  def spec = suite("Resource Tests")(
    test("use database") {
      for {
        conn <- ZIO.service[DbConnection]
        result <- conn.query("SELECT 1")
      } yield assertTrue(result == 1)
    }
  ).provide(
    DbConnection.layer  // Provide test dependencies
  )
}
```

### Using Before/After Patterns

```scala
import zio._
import zio.test._

object BeforeAfterSpec extends ZIOSpecDefault {
  def spec = {
    val testResource = ZIO.acquireRelease(
      ZIO.debug("Acquiring resource").as(new Resource())
    )(resource => ZIO.debug("Releasing resource").as(resource.close()))

    suite("Resource Management Tests")(
      test("use resource") {
        for {
          resource <- ZIO.service[Resource]
          result <- resource.use()
        } yield assertTrue(result)
      }
    ).provideSomeLayerShared(
      ZLayer.scoped(testResource.map(resource => Resource(resource)))
    )
  }

  class Resource {
    def use(): UIO[Boolean] = ZIO.succeed(true)
    def close(): Unit = ()
  }
}
```

## Mocking

ZIO Test supports mocking for isolated testing:

```scala
import zio._
import zio.test._
import zio.test.mock._

// Define a service
trait UserRepo {
  def getUser(id: String): Task[User]
  def saveUser(user: User): Task[Unit]
}

// Mock implementation
object UserRepoMock extends Mock[UserRepo] {
  object GetUser extends Effect[String, Throwable, User]
  object SaveUser extends Effect[User, Throwable, Unit]

  val compose: URLayer[Mock.Proxy, UserRepo] =
    ZLayer.fromFunction { proxy =>
      new UserRepo {
        def getUser(id: String): Task[User] = proxy(GetUser, id)
        def saveUser(user: User): Task[Unit] = proxy(SaveUser, user)
      }
    }
}

// Using mocks in tests
test("test with mock") {
  for {
    user <- UserService.getUser("123")
  } yield assertTrue(user.name == "Test User")
}.provide(
  UserRepoMock.GetUser(equalTo("123"), value(User("123", "Test User")))
)
```

## Best Practices

### Naming Conventions

- Use descriptive names for test suites and cases
- Group related tests in the same suite
- Use naming pattern: "should <expected behavior> when <conditions>"

```scala
suite("UserService")(
  test("should return user when user exists") {
    // Test code
  },
  test("should throw exception when user doesn't exist") {
    // Test code
  }
)
```

### Test Organization

- Structure tests to mirror your application
- Use nested suites for hierarchical organization
- Separate unit, integration, and acceptance tests

### Debugging Tests

- Use `ZIO.debug` to print values during test execution
- Apply the `diagnose` test aspect to get more detailed error reports
- Use `ZIO.trace` to include stack traces in failures

```scala
test("debugging example") {
  for {
    value <- ZIO.succeed(42)
    _ <- ZIO.debug(s"Value is: $value")
  } yield assertTrue(value == 42)
} @@ diagnose
```

### Managing Test Flakiness

- Use `nonFlaky` to run tests multiple times
- Apply `eventually` for tests that may need retries
- Set appropriate timeouts for asynchronous tests

```scala
test("no flakiness") {
  assertTrue(Random.nextIntBounded(10) >= 0)
} @@ nonFlaky
```

## Troubleshooting

### Common Issues

#### Tests Not Running

- Ensure your test class extends `ZIOSpecDefault`
- Check that your build tool is configured with ZIO Test

#### Timeouts

- Increase timeout duration for slow tests:
    
    ```scala
    test("slow test") {  // Test code} @@ timeout(2.minutes)
    ```
    

#### Resource Leaks

- Use `ZIO.acquireRelease` or `ZIO.scoped` for proper resource cleanup
- Make sure resources are closed in the release action

#### Asynchronous Test Failures

- Verify that all async operations are properly converted to ZIO effects
- Use `ZIO.fromFuture` for Future conversions

## Document History

|Version|Date|Changes|Author|
|---|---|---|---|
|0.1|2025-04-09|Initial AI-generated draft|AI|
|0.2|2025-04-09|Human review and corrections|J. Smith|

---

_This guide helps AI agents and developers effectively utilize ZIO Test for testing functional Scala applications. It promotes best practices and provides practical examples to ensure robust test coverage._