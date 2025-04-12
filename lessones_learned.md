---
status: draft
created: 2025-04-09
last_updated: 2025-04-09
version: "0.1"
---
> [!draft] Draft Document
> This document is an initial draft and may change significantly.
# Lessons Learned

This document captures key insights and lessons learned during the implementation of the Fio Bank integration. By documenting these challenges and their solutions, we can avoid similar issues in future work.

## ZIO Testing Framework Issues

### 1. ZIO Chunk Handling

**Problem**: When reading files with `Files.readAllBytes()`, it returns a `zio.Chunk[Byte]` that cannot be directly passed to `new String()`.

**Solution**: Convert the Chunk to an Array before creating a String:
```scala
// Incorrect
jsonString <- Files.readAllBytes(Path(exampleJsonFile)).map(new String(_))

// Correct
jsonBytes <- Files.readAllBytes(Path(exampleJsonFile))
jsonString = new String(jsonBytes.toArray)
```

## STTP Client Testing

### 1. Backend Stub Compatibility

**Problem**: `SyncBackendStub` cannot be directly used where `Backend[Task]` is expected.

**Solution**: We are using ZIO backend (which requires additional libraries), use HttpClientZioBackend.stub instead:

```scala
// Problematic approach
private val testBackend: Backend[Task] = SyncBackendStub
    .whenRequestMatches(...)
    .thenRespondAdjust(...)

// Better approach
private val testBackend: Backend[Task] = HttpClientZioBackend.stub
    .whenRequestMatches(...)
    .thenRespondAdjust(...)
```

## ZIO Environment and Reference Issues

### 1. Ref Creation in Tests

**Problem**: Creating a `Ref` in test code using `Ref.unsafe.make()` requires an implicit `Unsafe` parameter.

**Solution**: Use the implicit unsafe block to create Refs in test code:
```scala
// Incorrect
private val storage = Ref.unsafe.make(Map.empty[TransactionId, Transaction])

// Correct
private val storage: Ref[Map[TransactionId, Transaction]] =
    Unsafe.unsafely:
        Ref.unsafe.make(Map.empty[TransactionId, Transaction])
```

## Command-Line Interface Development

### 1. Command Line Argument Handling

**Problem**: Using `CommandLine.getArgs` directly can cause compilation errors.

**Solution**: In ZIO 2.x, access command line arguments through the `getArgs` method provided by `ZIOAppDefault`:
```scala
// Incorrect
args <- ZIO.succeed(CommandLine.getArgs)

// Correct
args <- getArgs
```

### 2. CLI Tool Testing

**Problem**: Manual testing of the CLI tool can be cumbersome.

**Solution**: Create a shell script wrapper that provides a more user-friendly interface and handles environment variables:
```bash
#!/bin/bash
# ...
SBT_COMMAND="sbtn \"fio/runMain works.iterative.incubator.fio.cli.FioCliMain $*\""
echo "Running: $SBT_COMMAND"
eval "$SBT_COMMAND"
```

## General Development Best Practices

### 1. Incremental Testing

**Problem**: Addressing multiple test issues at once can be overwhelming.

**Solution**: Fix tests one at a time, starting with the simplest ones. Run tests frequently to catch new issues early.

### 2. Adapt to Existing Patterns

**Problem**: Trying to force a different implementation style than what's used in the rest of the codebase.

**Solution**: Study the existing implementations thoroughly before starting. In our case, observing how `PostgreSQLTransactionRepository` was implemented provided valuable insights for our own implementations.

### 3. Domain-Driven Design Consistency

**Problem**: Inconsistent application of DDD principles can lead to architectural issues.

**Solution**: Regularly review bounded context boundaries and ensure that dependencies flow in the correct direction. Use classification comments to clarify the architectural role of each component.
