---
status: draft
created: 2025-04-15
last updated: 2025-04-15
version: "0.1"
---
> [!draft] Draft Document
> This document is an initial draft and may change significantly.

# Using Remote Docker Hosts with TestContainers in Java

## Overview

When working with containers that don't support Apple Silicon (like MS SQL Server), connecting to a remote Linux Docker host is an excellent solution. This approach lets you run x86_64 containers natively on a compatible system while developing on your Mac.

## Configuration Hierarchy

Java TestContainers loads configuration from multiple locations in the following priority order:

1. **Environment variables** - Must be in uppercase with underscores, preceded by `TESTCONTAINERS_` 
   (e.g., `checks.disable` becomes `TESTCONTAINERS_CHECKS_DISABLE`)
2. **`.testcontainers.properties` in user's home folder**
   - Linux: `/home/myuser/.testcontainers.properties`
   - Windows: `C:/Users/myuser/.testcontainers.properties`
   - macOS: `/Users/myuser/.testcontainers.properties`
3. **`testcontainers.properties` on the classpath** (e.g., in `src/test/resources`)

If multiple classpath configuration files exist, they are merged with the following precedence:
- Files on the 'local' classpath (URLs starting with `file:`) take priority
- Files in JAR dependencies are considered in alphabetical order of path

## Configuration Options

### Option 1: Configure TestContainers to Use a Remote Docker Host

TestContainers can be configured to use a remote Docker host through environment variables or a properties file.

#### Using Environment Variables

```bash
# Set environment variables before running your tests
export DOCKER_HOST=tcp://your-remote-host:2375
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
export TESTCONTAINERS_HOST_OVERRIDE=your-remote-host
```

In Java code, you can also set these programmatically:

```java
// Set system properties for the JVM
System.setProperty("DOCKER_HOST", "tcp://your-remote-host:2375");
System.setProperty("TESTCONTAINERS_HOST_OVERRIDE", "your-remote-host");

// Then create your container
MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
mysql.start();
```

#### Using Properties File

Create a file at `~/.testcontainers.properties` with:

```properties
# Use the EnvironmentAndSystemPropertyClientProviderStrategy for advanced configuration
docker.client.strategy=org.testcontainers.dockerclient.EnvironmentAndSystemPropertyClientProviderStrategy

# Configure Docker host connection (escape colons with backslashes)
docker.host=tcp\://193.86.200.14:2376
docker.tls.verify=true
docker.cert.path=/Users/username/.docker/contexts/tls/c7ccc9baf0d00a8200202d17675e1dd2437985795e149d36ef035c5c9542ae28/docker
```

Or alternatively, you can place this configuration in your project's `src/test/resources/testcontainers.properties` file to share it with your team.

### Option 2: Configure Docker Context

You can use Docker contexts to switch between local and remote Docker hosts:

```bash
# Create a new context pointing to your remote host
docker context create remote-linux --docker "host=ssh://username@your-remote-host"

# Switch to the remote context
docker context use remote-linux

# Verify connection
docker info
```

Note: Java Testcontainers library does not work directly with Docker contexts at this time. You'll need to use the configuration options above.

## Setting Up the Remote Linux Host

### 1. Install Docker on the Remote Host

```bash
# Update package index
sudo apt update

# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add Docker repository
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install Docker
sudo apt update
sudo apt install -y docker-ce

# Add your user to docker group to avoid using sudo
sudo usermod -aG docker ${USER}
```

### 2. Configure Docker to Accept Remote Connections

Edit the Docker service file:

```bash
sudo vim /lib/systemd/system/docker.service
```

Modify the `ExecStart` line to allow connections:

```
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375
```

Restart Docker service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 3. Set Up SSH for Easy Connection

From your Mac:

```bash
# Generate SSH key if you don't have one
ssh-keygen -t rsa

# Copy your SSH key to the remote host
ssh-copy-id username@your-remote-host

# Test connection
ssh username@your-remote-host
```

## Java TestContainers Examples

### Basic Java Example

```java
import org.testcontainers.containers.MSSQLServerContainer;
import org.junit.jupiter.api.Test;

class MSSQLTest {
    
    static {
        // Configure TestContainers to use remote Docker host
        System.setProperty("DOCKER_HOST", "tcp://your-remote-host:2375");
        System.setProperty("TESTCONTAINERS_HOST_OVERRIDE", "your-remote-host");
    }
    
    @Test
    void testWithSQL() {
        try (MSSQLServerContainer<?> mssql = new MSSQLServerContainer<>("mcr.microsoft.com/mssql/server:2022-latest")
                .withPassword("Strong.Password-123")
                .withInitScript("init-script.sql")) {
            
            mssql.start();
            
            // Get the mapped host and port
            String host = mssql.getHost(); // This will be your remote host, not localhost
            Integer port = mssql.getMappedPort(1433);
            
            String jdbcUrl = String.format("jdbc:sqlserver://%s:%d;databaseName=master", host, port);
            // Use the connection in your tests
        }
    }
}
```

### JUnit 5 Extension Example

```java
import org.junit.jupiter.api.extension.RegisterExtension;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class PostgresTest {
    
    static {
        // Configure TestContainers to use remote Docker host
        System.setProperty("DOCKER_HOST", "tcp://your-remote-host:2375");
        System.setProperty("TESTCONTAINERS_HOST_OVERRIDE", "your-remote-host");
    }
    
    @Container
    private PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:14")
            .withDatabaseName("test")
            .withUsername("test")
            .withPassword("test");
    
    @Test
    void testWithPostgres() {
        // The container is started automatically
        String jdbcUrl = postgres.getJdbcUrl();
        // Use postgres in your test
    }
}
```

## ZIO TestContainers Integration for Remote Hosts

Here's an example of configuring TestContainers with ZIO to use a remote Docker host:

```scala
object MSSQLCSNOLDataSourceSpec extends ZIOSpecDefault {
  
  // Set up environment variables for remote Docker in the test
  // This can also be done at the system level
  ZIO.attempt {
    System.setProperty("DOCKER_HOST", "tcp://your-remote-host:2375")
    System.setProperty("TESTCONTAINERS_HOST_OVERRIDE", "your-remote-host")
  }.orDie
  
  // Define Docker image 
  private val mssqlImage = DockerImageName.parse("mcr.microsoft.com/mssql/server:2022-latest")

  // ZIO Layer for the MSSQL container
  val mssqlContainer = ZLayer {
    ZIO.acquireRelease {
      ZIO.attempt {
        val container = MSSQLServerContainer.Def(
          dockerImageName = mssqlImage,
          password = "Strong.Password-123",
          urlParams = Map(),
          commonJdbcParams = JdbcDatabaseContainer.CommonParams()
        ).createContainer()
        
        container.start()
        
        // Initialize the database with the schema script
        val initScriptFile = Source.fromResource("db-init.sql").mkString
        val connection = DriverManager.getConnection(
          container.jdbcUrl, 
          container.username, 
          container.password
        )
        val statement = connection.createStatement()
        statement.execute(initScriptFile)
        statement.close()
        connection.close()
        
        container
      }
    }(container =>
      ZIO.attempt {
        container.container.stop()
      }.orDie
    )
  }
  
  // Rest of the test implementation...
}
```

## Network Considerations

When using a remote Docker host:

1. **Port Mapping**: TestContainers will automatically map the container ports to the remote host.

2. **Connection URL**: You need to ensure your application connects to `your-remote-host:mapped-port` rather than `localhost:mapped-port`.

3. **Container Networking**: Use the following approach to get the correct host and port:

```java
// Get the mapped host and port
String host = container.getHost();        // Returns the remote host address
Integer port = container.getMappedPort(1433);

// Use these for your connection string
String jdbcUrl = String.format("jdbc:sqlserver://%s:%d;databaseName=master", host, port);
```

## Security Considerations

When exposing Docker API remotely:

1. **Firewall Configuration**: Only allow connections from trusted IP addresses.

2. **TLS Authentication**: For production, configure Docker to use TLS certificates:

```bash
sudo dockerd --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem -H=0.0.0.0:2376
```

Then update your TestContainers configuration:

```properties
docker.client.strategy=org.testcontainers.dockerclient.EnvironmentAndSystemPropertyClientProviderStrategy
docker.host=tcp\://your-remote-host:2376
docker.tls.verify=true
docker.cert.path=/path/to/your/certs
```

3. **SSH Tunneling**: A more secure alternative is to use SSH tunneling instead of exposing the Docker API directly:

```bash
# On your Mac, create an SSH tunnel
ssh -L 2375:localhost:2375 username@your-remote-host

# Then use localhost:2375 as your DOCKER_HOST
```

## Disabling Features (Additional Configuration)

### Disable Startup Checks

Add to your properties file to speed up tests once everything is working:

```properties
checks.disable=true
```

Or via environment variable:

```bash
export TESTCONTAINERS_CHECKS_DISABLE=true
```

### Disable Ryuk Resource Reaper

If your environment already has container cleanup but doesn't allow privileged containers:

```bash
export TESTCONTAINERS_RYUK_DISABLED=true
```

## CI/CD Integration

This approach works well with CI/CD pipelines, as you can configure your CI system to use a dedicated Docker host for running tests.

### GitHub Actions Example

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up JDK
        uses: actions/setup-java@v2
        with:
          java-version: '17'
          distribution: 'adopt'
      - name: Run tests
        run: ./gradlew test
        env:
          DOCKER_HOST: tcp://docker-host:2375
          TESTCONTAINERS_HOST_OVERRIDE: docker-host
```

## Conclusion

Using a remote Docker host provides:

1. **Full Compatibility**: Run any container regardless of architecture
2. **Better Performance**: Native execution instead of emulation
3. **Resource Isolation**: Heavy containers run on a separate machine

This approach is ideal for development teams with mixed architectures or when working with containers that don't support ARM64 architecture.