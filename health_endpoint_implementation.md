---
status: stable
created: 2023-10-27
last updated: 2023-10-27
version: "1.0"
---
> [!stable] Stable Document
> This document defines our standards for implementing health endpoints in services.

# Health Endpoint Implementation Guide

## Overview
Health endpoints are crucial for monitoring operational status of services. They should check all critical components and return a consolidated health status that monitoring tools, load balancers, and orchestration platforms can use to make operational decisions.

## Status Levels

All health endpoints must support three status levels:

1. **UP** - Component is fully operational
2. **WARN** - Component is operational but requires attention (e.g., certificate expiring soon)
3. **DOWN** - Component is not operational

## Required Components to Check

All services should check at minimum:

1. **Database Connections** 
   - Connection pools
   - Basic query operations
   - Consider checking both read and write operations if applicable

2. **External Dependencies**
   - APIs/services that are critical to the operation
   - Messaging systems
   - Cache services

3. **Certificates and Authentication**
   - For services using certificates:
     - Certificate validity (including expiration warnings)
     - Authentication subsystems

4. **Resource Usage** (optional but recommended)
   - JVM heap usage
   - Thread pool saturation
   - Disk space availability

## Standard Health Status Model

```scala
case class HealthStatus(
    status: String,               // "UP", "WARN", or "DOWN"
    version: String,              // Service version
    components: Map[String, ComponentStatus]
)

case class ComponentStatus(
    status: String,               // "UP", "WARN", or "DOWN"
    details: Option[String]       // Detailed status information
)
```

## Warning Thresholds

Implement early warning thresholds for issues that can be detected before they become critical:

- Certificate expiration: Warn 30 days before expiration
- Disk space: Warn at 80% full
- Connection pool: Warn at 70% utilization
- Response time: Warn when approaching timeout thresholds

## Implementation Best Practices

### Caching Results

To avoid expensive health checks on every request:

- Cache results for an appropriate duration (e.g., 5 minutes for a certificate check)
- Use ZIO Ref or similar mechanism for functional caching
- Different components may have different cache durations

### HTTP Status Codes

- Return HTTP 200 (OK) for both UP and WARN status
- Return HTTP 503 (Service Unavailable) for DOWN status
- Include detailed status in response body regardless of HTTP status

### Proper Error Handling

- Always use timeouts for external calls
- Handle exceptions gracefully with detailed information
- Fail fast - health checks should complete quickly

### Testing Capabilities

- For components that write/modify data, use lightweight test operations
- For signing operations, use a small test document
- For validation operations, test with predictable data

## Example Response

```json
{
  "status": "WARN",
  "version": "1.2.3-b456",
  "components": {
    "database": {
      "status": "UP",
      "details": "Connected to certification_db, 3/20 active connections"
    },
    "certificate": {
      "status": "WARN",
      "details": "WARNING: Certificate will expire in 25 days"
    },
    "signing": {
      "status": "UP",
      "details": "Signing check successful at 2023-10-27T14:22:31Z"
    },
    "upstreamCA": {
      "status": "UP",
      "details": "CA service status: STARTED for CA aca3-2"
    }
  }
}
```

## Integration with Monitoring Systems

### Zabbix Configuration

For Zabbix, create triggers for:

1. HTTP status code (>= 500 for critical alerts)
2. JSON response path `$.status` equals "DOWN" for critical alerts
3. JSON response path `$.status` equals "WARN" for warning alerts
4. JSON response path `$.components.*` to monitor individual components

### Kubernetes Configuration

For Kubernetes liveness and readiness probes:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3
```

### Nomad/Consul Integration

To integrate health checks with our Nomad/Consul environment, add a `check` block to the service definition in the Nomad job file. Here's how to update our existing certification-extval service configuration:

```hcl
service {
  name = "certification-extval"
  port = "extvalhttp"
  
  check {
    name     = "health"
    type     = "http"
    path     = "/health"
    interval = "30s"
    timeout  = "5s"
    
    # Define what constitutes success (default is 200-299)
    success_before_passing = 2
    failures_before_critical = 3
    
    # Advanced configuration for checking JSON response content
    header {
      Accept = ["application/json"]
    }
    
    # Optional: Restart on repeated failures
    check_restart {
      limit = 3
      grace = "90s"
      ignore_warnings = true
    }
  }
}
```

Applying this to our specific deployment files (`staging_certification_service_extval.nomad` and `prod_certification_service_extval.nomad`), we would add the check block to the existing service definition.

This configuration will:
1. Check the `/health` endpoint every 30 seconds
2. Mark the service as healthy only after 2 consecutive successful responses
3. Mark the service as unhealthy after 3 consecutive failures
4. Request JSON response format specifically
5. Optionally restart the service after 3 consecutive failures with a 90-second grace period
6. Ignore warning states (only restart on critical failures)

## Common Health Check Implementations

### Database Health Check

Test query should be lightweight but verify real connectivity:

```scala
def checkDatabase: Task[ComponentStatus] =
  repository.isConnected
    .map(isConnected => // Convert to ComponentStatus)
    .catchAll(err => // Handle errors and timeouts)
```

### Certificate Health Check

Check validation period and implement warnings:

```scala
def checkCertificate: Task[ComponentStatus] =
  signingService.checkCertificateStatus.flatMap(_.details).map { certStatus =>
    // Convert CertificateStatusLevel to ComponentStatus
  }
```

### External API Health Check

Test connectivity with appropriate timeouts:

```scala
def checkExternalApi: Task[ComponentStatus] =
  externalService.status
    .timeoutFail(new TimeoutException("Connection timed out"))(5.seconds)
    .map(status => // Convert to ComponentStatus)
    .catchAll(err => // Handle errors)
```

## Internal Implementation

See the certification service's `LiveHealthRoutes` implementation for an example of a complete health endpoint using ZIO and Tapir.
