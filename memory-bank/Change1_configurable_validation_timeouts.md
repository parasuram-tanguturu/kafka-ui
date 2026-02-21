# Configurable Validation Timeouts

> YAML-configurable AdminClient timeout and retry settings for cluster connection validation.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Root Cause Analysis](#root-cause-analysis)
- [Solution Overview](#solution-overview)
- [Benefits](#benefits)
- [Architecture](#architecture)
- [Configuration Reference](#configuration-reference)
  - [Global Defaults](#global-defaults)
  - [Per-Cluster Override](#per-cluster-override)
  - [Full Example](#full-example)
- [Implementation Details](#implementation-details)
  - [ClustersProperties.java](#1-clusterspropertiesjava)
  - [KafkaClusterFactory.java](#2-kafkaclusterfactoryjava)
  - [KafkaServicesValidation.java](#3-kafkaservicesvalidationjava)
- [Data Flow](#data-flow)
- [Fallback Chain](#fallback-chain)
- [Use Cases](#use-cases)
- [Troubleshooting](#troubleshooting)
- [Related Files](#related-files)

---

## Problem Statement

When connecting to **managed Kafka services** (like Confluent Cloud, AWS MSK, etc.), users experienced persistent validation failures:

```
TimeoutException: Timed out waiting to send the call. Call: fetchMetadata
```

The validation logic used **hardcoded** timeout values that were too aggressive for cloud environments:

| Parameter | Original Value | Issue |
|-----------|----------------|-------|
| `retries` | 1 | Single retry insufficient for network variability |
| `request.timeout.ms` | 5000ms | Too short for cloud metadata responses |
| `default.api.timeout.ms` | 5000ms | Too short for API operations |

---

## Root Cause Analysis

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY VALIDATION FAILS FOR CLOUD CLUSTERS                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   LOCAL KAFKA                        CONFLUENT CLOUD                         │
│   ───────────                        ───────────────                         │
│   • Same machine/network             • Internet latency (50-200ms)           │
│   • Instant DNS resolution           • Cloud DNS resolution delays           │
│   • Direct TCP connection            • TLS handshake overhead                │
│   • No load balancer                 • Load balancer in path                 │
│   • Immediate metadata               • Distributed metadata retrieval        │
│                                                                              │
│   5 second timeout = ✅ OK           5 second timeout = ❌ FAILS             │
│                                                                              │
│   The UI would show "Error connecting to cluster" even though:              │
│   • Credentials were correct                                                 │
│   • Network was reachable                                                    │
│   • Background stats collection worked (using different timeouts)            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Two Separate Code Paths

| Path | AdminClient | Timeout Config | Purpose |
|------|-------------|----------------|---------|
| **Validation** | `kui-admin-client-validation-*` | Was hardcoded (now configurable) | UI "test connection" button |
| **Runtime Stats** | `kui-admin-client-*` | Uses `kafka.adminClientTimeout` | Background statistics collection |

The **validation path** had overly aggressive timeouts, causing false-negative connection failures.

---

## Solution Overview

Move validation timeout/retry values from hardcoded constants into **YAML configuration** with:

1. **Global defaults** - Applied to all clusters
2. **Per-cluster overrides** - Optional cluster-specific settings
3. **Code fallbacks** - Sensible defaults if not configured

```yaml
kafka:
  # Global defaults for all clusters
  validation:
    retries: 3
    requestTimeoutMs: 30000
    defaultApiTimeoutMs: 30000
  
  clusters:
    - name: local
      # Uses global defaults
      
    - name: confluent-cloud
      # Override for slower cloud connections
      validation:
        retries: 5
        requestTimeoutMs: 60000
        defaultApiTimeoutMs: 60000
```

---

## Benefits

| Benefit | Description |
|---------|-------------|
| **Cloud Compatibility** | Managed Kafka services work out-of-the-box with increased default timeouts |
| **Environment Flexibility** | Different settings for dev, staging, production |
| **Per-Cluster Tuning** | Slow clusters can have higher timeouts without affecting fast ones |
| **No Code Changes** | Operators can tune timeouts via YAML without rebuilding |
| **Backward Compatible** | Existing deployments work unchanged (sensible defaults applied) |
| **Debuggability** | Timeout values visible in configuration, not buried in code |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CONFIGURATION HIERARCHY                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     application.yml / application-local.yml          │   │
│   │  ─────────────────────────────────────────────────────────────────  │   │
│   │  kafka:                                                              │   │
│   │    validation:              ◄── Global defaults                     │   │
│   │      retries: 3                                                      │   │
│   │      requestTimeoutMs: 30000                                         │   │
│   │      defaultApiTimeoutMs: 30000                                      │   │
│   │    clusters:                                                         │   │
│   │      - name: confluent-cloud                                         │   │
│   │        validation:          ◄── Per-cluster override (optional)     │   │
│   │          retries: 5                                                  │   │
│   │          requestTimeoutMs: 60000                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     ClustersProperties.java                          │   │
│   │  ─────────────────────────────────────────────────────────────────  │   │
│   │  • ValidationConfig class with retries, requestTimeoutMs, etc.      │   │
│   │  • Global: kafka.validation                                          │   │
│   │  • Per-cluster: kafka.clusters[n].validation                         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     KafkaClusterFactory.java                         │   │
│   │  ─────────────────────────────────────────────────────────────────  │   │
│   │  resolveEffectiveValidationConfig():                                │   │
│   │    1. Check cluster.validation → if present, use it                 │   │
│   │    2. Else check kafka.validation → if present, use it              │   │
│   │    3. Else use code defaults (3, 30000, 30000)                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   KafkaServicesValidation.java                       │   │
│   │  ─────────────────────────────────────────────────────────────────  │   │
│   │  validateClusterConnection(..., retries, requestTimeoutMs,          │   │
│   │                             defaultApiTimeoutMs)                     │   │
│   │  ↳ Applies settings to AdminClient configuration                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     Kafka AdminClient                                │   │
│   │  ─────────────────────────────────────────────────────────────────  │   │
│   │  AdminClientConfig.RETRIES_CONFIG = retries                         │   │
│   │  AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG = requestTimeoutMs     │   │
│   │  AdminClientConfig.DEFAULT_API_TIMEOUT_MS_CONFIG = defaultApiTimeout│   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Configuration Reference

### Global Defaults

Set under `kafka.validation` to apply to all clusters:

```yaml
kafka:
  validation:
    retries: 3                    # Number of AdminClient retries
    requestTimeoutMs: 30000       # request.timeout.ms (milliseconds)
    defaultApiTimeoutMs: 30000    # default.api.timeout.ms (milliseconds)
```

### Per-Cluster Override

Set under each cluster's `validation` block to override global defaults:

```yaml
kafka:
  clusters:
    - name: my-slow-cluster
      bootstrapServers: broker.example.com:9092
      validation:
        retries: 5
        requestTimeoutMs: 60000
        defaultApiTimeoutMs: 60000
```

### Full Example

```yaml
kafka:
  # Global validation settings (applied to all clusters by default)
  validation:
    retries: 3
    requestTimeoutMs: 30000
    defaultApiTimeoutMs: 30000

  clusters:
    # Local development - uses global defaults
    - name: local
      bootstrapServers: localhost:9092
      # No validation block → uses global defaults

    # Confluent Cloud - needs higher timeouts
    - name: confluent-cloud
      bootstrapServers: pkc-xxx.us-central1.gcp.confluent.cloud:9092
      properties:
        security.protocol: SASL_SSL
        sasl.mechanism: PLAIN
        sasl.jaas.config: >
          org.apache.kafka.common.security.plain.PlainLoginModule required
          username='API_KEY' password='API_SECRET';
      validation:
        retries: 5
        requestTimeoutMs: 60000
        defaultApiTimeoutMs: 60000

    # AWS MSK - moderate timeout increase
    - name: aws-msk
      bootstrapServers: b-1.msk-cluster.region.amazonaws.com:9092
      validation:
        retries: 4
        requestTimeoutMs: 45000
        defaultApiTimeoutMs: 45000
```

---

## Implementation Details

### 1. ClustersProperties.java

**Added `ValidationConfig` inner class:**

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public static class ValidationConfig {
  Integer retries = 3;              // Default: 3 retries
  Integer requestTimeoutMs = 30000; // Default: 30 seconds
  Integer defaultApiTimeoutMs = 30000;
}
```

**Added global validation field:**

```java
ValidationConfig validation = new ValidationConfig();
```

**Added per-cluster validation field in `Cluster` class:**

```java
ValidationConfig validation;  // Optional - null if not specified
```

### 2. KafkaClusterFactory.java

**Added `ClustersProperties` dependency** to access global config:

```java
private final ClustersProperties clustersProperties;

public KafkaClusterFactory(WebclientProperties webclientProperties,
                           JmxMetricsRetriever jmxMetricsRetriever,
                           ClustersProperties clustersProperties) {
  // ...
  this.clustersProperties = clustersProperties;
}
```

**Added `resolveEffectiveValidationConfig()` method:**

```java
private ClustersProperties.ValidationConfig resolveEffectiveValidationConfig(
    ClustersProperties.Cluster clusterProperties) {
  ClustersProperties.ValidationConfig globalDefaults = clustersProperties.getValidation();
  ClustersProperties.ValidationConfig clusterOverride = clusterProperties.getValidation();

  int retries = Optional.ofNullable(clusterOverride)
      .map(ClustersProperties.ValidationConfig::getRetries)
      .orElseGet(() -> Optional.ofNullable(globalDefaults)
          .map(ClustersProperties.ValidationConfig::getRetries)
          .orElse(3));

  // Similar for requestTimeoutMs and defaultApiTimeoutMs...
  
  return new ClustersProperties.ValidationConfig(retries, requestTimeoutMs, defaultApiTimeoutMs);
}
```

**Updated `validate()` method** to pass effective settings:

```java
ClustersProperties.ValidationConfig effectiveValidation = 
    resolveEffectiveValidationConfig(clusterProperties);

validateClusterConnection(
    clusterProperties.getBootstrapServers(),
    convertProperties(clusterProperties.getProperties()),
    clusterProperties.getSsl(),
    effectiveValidation.getRetries(),
    effectiveValidation.getRequestTimeoutMs(),
    effectiveValidation.getDefaultApiTimeoutMs()
)
```

### 3. KafkaServicesValidation.java

**Changed method signature** to accept timeout parameters:

```java
// Before (hardcoded)
public static Mono<ApplicationPropertyValidationDTO> validateClusterConnection(
    String bootstrapServers,
    Properties clusterProps,
    @Nullable TruststoreConfig ssl)

// After (configurable)
public static Mono<ApplicationPropertyValidationDTO> validateClusterConnection(
    String bootstrapServers,
    Properties clusterProps,
    @Nullable TruststoreConfig ssl,
    int retries,
    int requestTimeoutMs,
    int defaultApiTimeoutMs)
```

**Uses parameters instead of hardcoded values:**

```java
properties.put(AdminClientConfig.RETRIES_CONFIG, retries);
properties.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, requestTimeoutMs);
properties.put(AdminClientConfig.DEFAULT_API_TIMEOUT_MS_CONFIG, defaultApiTimeoutMs);
```

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VALIDATION REQUEST FLOW                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User clicks "Test Connection" in UI                                        │
│              │                                                               │
│              ▼                                                               │
│  ┌───────────────────────┐                                                  │
│  │ ApplicationConfigController                                              │
│  │ validateConfig()      │                                                  │
│  └───────────┬───────────┘                                                  │
│              │                                                               │
│              ▼                                                               │
│  ┌───────────────────────┐                                                  │
│  │ KafkaClusterFactory   │                                                  │
│  │ validate()            │                                                  │
│  │  ↳ resolveEffective   │◄── Computes effective validation config         │
│  │    ValidationConfig() │    (cluster override > global > defaults)        │
│  └───────────┬───────────┘                                                  │
│              │                                                               │
│              ▼                                                               │
│  ┌───────────────────────┐                                                  │
│  │ KafkaServicesValidation                                                  │
│  │ validateClusterConnection(                                               │
│  │   ..., retries=3,     │◄── Uses effective settings                      │
│  │   requestTimeout=30000,│                                                 │
│  │   apiTimeout=30000)   │                                                  │
│  └───────────┬───────────┘                                                  │
│              │                                                               │
│              ▼                                                               │
│  ┌───────────────────────┐                                                  │
│  │ Kafka AdminClient     │                                                  │
│  │ listTopics().names()  │◄── AdminClient uses configured timeouts         │
│  └───────────┬───────────┘                                                  │
│              │                                                               │
│              ▼                                                               │
│  ┌───────────────────────┐                                                  │
│  │ Response to UI        │                                                  │
│  │ ✅ Valid / ❌ Error    │                                                  │
│  └───────────────────────┘                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Fallback Chain

The effective validation config is resolved using a **priority-based fallback chain**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FALLBACK PRIORITY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Priority 1 (Highest): Per-Cluster Override                                │
│   ──────────────────────────────────────                                    │
│   kafka.clusters[n].validation.retries                                      │
│                                                                              │
│           │ if null                                                          │
│           ▼                                                                  │
│                                                                              │
│   Priority 2: Global Defaults                                               │
│   ───────────────────────────                                               │
│   kafka.validation.retries                                                   │
│                                                                              │
│           │ if null                                                          │
│           ▼                                                                  │
│                                                                              │
│   Priority 3 (Lowest): Code Defaults                                        │
│   ──────────────────────────────────                                        │
│   retries = 3                                                                │
│   requestTimeoutMs = 30000                                                   │
│   defaultApiTimeoutMs = 30000                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Use Cases

### 1. Confluent Cloud

```yaml
kafka:
  clusters:
    - name: confluent-cloud
      bootstrapServers: test-xxx.confluent.cloud:9092
      properties:
        security.protocol: SASL_SSL
        sasl.mechanism: PLAIN
        sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username='KEY' password='SECRET';
      validation:
        retries: 5
        requestTimeoutMs: 60000
        defaultApiTimeoutMs: 60000
```

### 2. High-Latency Network

```yaml
kafka:
  validation:
    retries: 5
    requestTimeoutMs: 120000
    defaultApiTimeoutMs: 120000
```

### 3. Fast Local Development

```yaml
kafka:
  validation:
    retries: 1
    requestTimeoutMs: 5000
    defaultApiTimeoutMs: 5000
```

---

## Troubleshooting

### Validation Still Failing

**Check the logs for actual timeout values:**

Look for `AdminClientConfig values:` in logs to verify settings are applied:

```
AdminClientConfig values:
    ...
    default.api.timeout.ms = 30000
    retries = 3
    request.timeout.ms = 30000
```

**Increase timeouts further:**

For very slow networks, try:

```yaml
kafka:
  clusters:
    - name: slow-cluster
      validation:
        retries: 10
        requestTimeoutMs: 120000
        defaultApiTimeoutMs: 120000
```

### Config Not Being Applied

**Verify YAML indentation:**

```yaml
# CORRECT
kafka:
  validation:
    retries: 5

# WRONG (nested under clusters incorrectly)
kafka:
  clusters:
    validation:  # This won't work!
      retries: 5
```

**Check Spring profile:**

Ensure you're using the correct profile:

```bash
./gradlew :api:bootRun --args='--spring.profiles.active=local'
```

---

## Related Files

| File | Purpose |
|------|---------|
| `api/src/main/java/io/kafbat/ui/config/ClustersProperties.java` | Configuration model with `ValidationConfig` class |
| `api/src/main/java/io/kafbat/ui/service/KafkaClusterFactory.java` | Resolves effective config and calls validation |
| `api/src/main/java/io/kafbat/ui/util/KafkaServicesValidation.java` | Applies settings to AdminClient |
| `api/src/main/resources/application-local.yml` | Example with global and per-cluster config |
| `api/src/main/resources/application-dev.yml` | Example with global config |

---

## Debug Session Summary

This feature was implemented after debugging Confluent Cloud connection failures:

1. **Symptom**: UI showed "Error connecting to cluster" despite valid credentials
2. **Investigation**: Background stats worked (using different AdminClient settings), but validation failed
3. **Root Cause**: Validation AdminClient used hardcoded aggressive timeouts (5s, 1 retry)
4. **Initial Fix**: Increased hardcoded values to (30s, 3 retries)
5. **Enhancement**: Made timeouts YAML-configurable for flexibility

The solution ensures managed Kafka services work reliably while allowing operators to tune settings for their specific environments.
