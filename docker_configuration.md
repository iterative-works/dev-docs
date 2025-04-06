---
status: draft
last updated: 2025-04-06
version: "0.1"
---
> [!draft] Draft Document
> This document is an initial draft and may change significantly.
# Docker Configuration for IW Incubator

This document details how we've configured Docker for the IW Incubator project, specifically for the YNAB Importer application. 

## Overview

We use `sbt-native-packager` to build a Docker image of our application. This enables:

1. Consistent deployment across environments
2. Integration with TestContainers for end-to-end testing
3. Simplified deployment to production or staging environments

## Configuration Details

### Plugin Setup

In `project/plugins.sbt`, we add the native packager plugin:

```scala
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.11.1")
```

### Docker Configuration in build.sbt

We've configured Docker for the root project, since it contains the main application launcher. Here's the full configuration:

```scala
.enablePlugins(VitePlugin, JavaServerAppPackaging, DockerPlugin, IWScalaProjectPlugin)
// ...
.settings(
    // Docker configuration
    Docker / packageName := "iw-incubator",
    dockerBaseImage := "eclipse-temurin:21-jre-alpine",
    dockerExposedPorts := Seq(8080),
    dockerUpdateLatest := true,
    dockerEnvVars := Map(
        "BLAZE_HOST" -> "0.0.0.0",
        "BLAZE_PORT" -> "8080",
        "BASEURI" -> "/",
        "VITE_FILE" -> "/opt/docker/vite/manifest.json",
        "JAVA_OPTS" -> "-Xmx512m -Xms256m",
        "LOG_LEVEL" -> "INFO"
    ),
    // Add healthcheck
    dockerCommands += Cmd(
        "HEALTHCHECK",
        "--interval=30s",
        "--timeout=5s",
        "--start-period=30s",
        "--retries=3",
        "CMD",
        "curl",
        "-f",
        "http://localhost:8080/health",
        "||",
        "exit",
        "1"
    )
)
```

Key aspects of this configuration include:

1. **Base Image**: `eclipse-temurin:21-jre-alpine` - A lightweight Alpine-based Java 21 JRE
2. **Exposed Ports**: Port 8080 is exposed for HTTP traffic
3. **Environment Variables**:
   - `BLAZE_HOST` and `BLAZE_PORT`: Configure the HTTP4S Blaze server to bind to all interfaces
   - `BASEURI`: Root path for application URLs
   - `VITE_FILE`: Location of the Vite manifest for static assets
   - `JAVA_OPTS`: JVM memory configuration
   - `LOG_LEVEL`: Application logging level
4. **Health Check**: A health endpoint at `/health` is configured to verify application readiness

## Health Endpoint

We've implemented a dedicated health endpoint in `HealthModule.scala` that:

1. Checks database connectivity
2. Returns HTTP 200 when all systems are operational
3. Returns HTTP 503 when any dependencies are unavailable
4. Returns JSON with detailed status information

The health endpoint is used by:
- Docker's HEALTHCHECK directive to monitor container health
- TestContainers' wait strategy to determine when the application is ready for testing

## TestContainers Integration

The Docker image is used in our end-to-end testing framework with TestContainers. The `TestContainersSupport` class:

1. Creates a private Docker network
2. Launches a PostgreSQL container
3. Launches our application container with proper configuration
4. Provides test code with connection details for both containers

## Building the Docker Image

To build the Docker image:

```bash
sbt docker:publishLocal
```

This creates an image named `iw-incubator:0.1.0-SNAPSHOT` in your local Docker registry.

## Running the Docker Image

To run the image (assuming a PostgreSQL instance is available):

```bash
docker run -p 8080:8080 \
  -e PG_URL=jdbc:postgresql://host.docker.internal:5432/incubator \
  -e PG_USERNAME=incubator \
  -e PG_PASSWORD=your_password \
  iw-incubator:0.1.0-SNAPSHOT
```

For production deployments, additional environment variables may be needed:
- `YNAB_API_TOKEN`: For YNAB API integration
- `AUTH_SECRET`: For authentication (when implemented)

## Future Improvements

1. **Multi-stage builds**: Optimize the Docker image size further
2. **Secrets management**: Improve handling of sensitive values like API tokens
3. **Separate profiles**: Create different Docker configurations for development, testing, and production
4. **CI/CD integration**: Automate Docker build and publishing as part of our CI pipeline