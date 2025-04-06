---
status: in-review
last updated: 2025-04-06
version: "0.1"
---
> [!review] In-Review Document
> This document is under review and feedback is welcome.
# TestContainers with ZIO Integration

## Setup Overview

Our integration tests use TestContainers to provide a PostgreSQL database in a Docker container. We integrate this with ZIO to manage the lifecycle and provide the database as a ZIO layer for tests.

### Key Components

1. **PostgreSQLLayers** - Provides ZIO layers for:
   - PostgreSQL container management
   - DataSource configuration
   - Flyway migration service
   - Repository implementations

2. **MigrateAspects** - Provides a test aspect that:
   - Cleans the database before each test suite
   - Runs migrations to create a fresh schema
   - Can be applied to test suites with `@@ migrate`

3. **Sequential Testing** - All tests use `@@ sequential` to prevent concurrent database access issues

### Implementation Details

#### Container Setup

```scala
// 1. Define the Docker image
private val postgresImage = DockerImageName.parse("postgres:17-alpine")

// 2. Create a ZIO layer that manages container lifecycle
val postgresContainer = ZLayer {
    ZIO.acquireRelease {
        ZIO.attempt {
            val container = PostgreSQLContainer(postgresImage)
            container.start()
            container
        }
    }(container =>
        ZIO.attempt {
            container.stop()
        }.orDie
    )
}
```

#### DataSource Configuration

```scala
// Create a DataSource from the container
val dataSourceLayer = postgresContainer >>> ZLayer.fromZIO {
    for
        container <- ZIO.service[PostgreSQLContainer]
        _ <- ZIO.attempt(Class.forName("org.postgresql.Driver"))
        dataSource <- ZIO.acquireRelease(ZIO.attempt {
            val config = new HikariConfig()
            config.setJdbcUrl(container.jdbcUrl)
            config.setUsername(container.username)
            config.setPassword(container.password)
            config.setMaximumPoolSize(5)
            new HikariDataSource(config)
        })(dataSource => ZIO.attempt(dataSource.close()).orDie)
    yield dataSource
}
```

#### Database Migration

```scala
// Create a custom Flyway config that allows cleaning the database
val testFlywayConfig = FlywayConfig(
    locations = FlywayConfig.DefaultLocation :: Nil,
    cleanDisabled = false
)

// Test aspect that handles database setup
val migrate = TestAspect.before(
    ZIO.scoped {
        for
            migrationService <- ZIO.service[FlywayMigrationService]
            _ <- migrationService.clean()
            result <- migrationService.migrate()
        yield ()
    }
)
```

#### Test Suite Integration

```scala
def spec = {
    suite("PostgreSQLRepositorySpec")(
        // Test cases...
    ) @@ sequential @@ migrate
}.provideSomeShared[Scope](
    flywayMigrationServiceLayer,
    repositoryLayer
)
```

### Benefits

1. **Clean State** - Each test suite runs against a fresh database schema
2. **Isolation** - Tests don't interfere with each other
3. **Realistic Testing** - Tests run against the actual PostgreSQL database, not mocks
4. **Reproducibility** - Tests are consistent across environments

## Dependencies

Add these dependencies to your project:

```scala
"com.dimafeng" %% "testcontainers-scala-postgresql" % "0.40.15" % Test,
"org.testcontainers" % "postgresql" % "1.18.0" % Test,
"org.flywaydb" % "flyway-core" % "9.21.1" % Test
```

## Required Imports

```scala
import zio.*
import zio.test.*
import com.dimafeng.testcontainers.PostgreSQLContainer
import org.testcontainers.utility.DockerImageName
import com.zaxxer.hikari.{HikariConfig, HikariDataSource}
import javax.sql.DataSource
import zio.test.TestAspect.sequential
```
# ## Remote Docker Configuration
## Remote Docker Configuration

When running TestContainers in CI environments or with remote Docker hosts, additional configuration is required.

> [!IMPORTANT]
> TestContainers does not use Docker contexts and doesn't support SSH protocol for remote connections.

### Remote Connection Options

TestContainers can be configured to use a remote Docker host in three ways:

#### 1. Properties File

Create a file at `$HOME/.testcontainers.properties`:

```properties
testcontainers.docker.host=tcp://remote-docker-host:2376
testcontainers.docker.tls.verify=true
testcontainers.docker.cert.path=/path/to/docker/certs
```

#### 2. JVM System Properties

Add the following options when running your application:

```
-Dtestcontainers.docker.host=tcp://remote-docker-host:2376
-Dtestcontainers.docker.tls.verify=true
-Dtestcontainers.docker.cert.path=/path/to/docker/certs
```

#### 3. Environment Variables

Set standard Docker environment variables:

```bash
DOCKER_HOST=tcp://remote-docker-host:2376
DOCKER_TLS_VERIFY=1
DOCKER_CERT_PATH=/path/to/docker/certs
```

### TLS Certificate Requirements

When using a TLS-enabled Docker host, the certificate path must contain:

- `ca.pem` - Certificate Authority certificate
- `cert.pem` - Client certificate
- `key.pem` - Client private key

### Example Configuration

```properties
# Example remote connection to a Docker host
testcontainers.docker.host=tcp://193.86.200.14:2376
testcontainers.docker.tls.verify=true
testcontainers.docker.cert.path=/Users/mph/.docker/contexts/tls/c7ccc9baf0d00a8200202d17675e1dd2437985795e149d36ef035c5c9542ae28
```

### CI Implementation

For CI environments:

1. Store Docker certificates as encrypted secrets
2. Extract certificates to the appropriate path during job initialization
3. Set the necessary environment variables
4. Run tests which will connect to the remote Docker host

### Troubleshooting

If you encounter connection issues:

1. Ensure the Docker daemon is running on the remote host
2. Verify the TCP endpoint is enabled and accessible
3. Check that TLS certificates are valid and properly configured
4. Confirm firewall rules allow connections on the Docker port (usually 2376)
5. Use `docker -H tcp://host:port info` to test direct connection outside TestContainers