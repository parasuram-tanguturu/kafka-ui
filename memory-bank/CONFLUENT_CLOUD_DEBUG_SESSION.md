# Confluent Cloud Debugging Session

> Session documenting the resolution of two distinct failure modes when connecting Kafka UI to Confluent Cloud.

---

## Table of Contents

- [Problem Overview](#problem-overview)
- [Issue 1: Runtime Stats Quorum Failure](#issue-1-runtime-stats-quorum-failure)
- [Issue 2: Validation Timeout False Negatives](#issue-2-validation-timeout-false-negatives)
- [Files Modified](#files-modified)
- [Key Insight](#key-insight)
- [Debugging Evidence Trail](#debugging-evidence-trail)

---

## Problem Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SAME CREDENTIALS                                     │
│                              │                                               │
│         ┌────────────────────┴────────────────────┐                         │
│         ▼                                         ▼                         │
│  ┌─────────────────────┐                ┌─────────────────────┐             │
│  │  RUNTIME STATS PATH │                │  VALIDATION PATH    │             │
│  │  (long-lived client)│                │  (short-lived client)│            │
│  └──────────┬──────────┘                └──────────┬──────────┘             │
│             │                                      │                         │
│   ┌─────────┴─────────┐                  ┌─────────┴─────────┐              │
│   │ describeCluster ✓ │                  │ listTopics        │              │
│   │ describeQuorum  ✗ │◄── RESTRICTED    │ + tight timeouts  │◄── TIMEOUT  │
│   └───────────────────┘    API           └───────────────────┘    ISSUES    │
│                                                                              │
│  Result: Partial failure    Result: False "connection error"                │
│  (quorum throws exception)  (even when cluster is healthy)                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Core Insight:**

```
SAME CREDENTIALS + DIFFERENT API CALL + DIFFERENT TIMEOUT POLICY = DIFFERENT OUTCOMES
```

---

## Issue 1: Runtime Stats Quorum Failure

### WHY

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Confluent Cloud Managed Kafka                                              │
│   ┌────────────────────────────────────────┐                                │
│   │  describeMetadataQuorum() API          │                                │
│   │  ══════════════════════════════════    │                                │
│   │  • Requires special permissions        │                                │
│   │  • Not exposed on managed clusters     │                                │
│   │  • Throws ClusterAuthorizationException│                                │
│   └────────────────────────────────────────┘                                │
│                        │                                                     │
│                        ▼                                                     │
│   ┌────────────────────────────────────────┐                                │
│   │  StatisticsService.loadQuorumInfo()    │                                │
│   │  ════════════════════════════════════  │                                │
│   │  OLD: Exception = CLUSTER FAILURE      │ ◄── Wrong behavior             │
│   │  NEW: Exception = No quorum info       │ ◄── Correct behavior           │
│   └────────────────────────────────────────┘                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### HOW

```
                     BEFORE FIX                          AFTER FIX
                 ┌───────────────┐                   ┌───────────────┐
                 │ loadQuorumInfo│                   │ loadQuorumInfo│
                 └───────┬───────┘                   └───────┬───────┘
                         │                                   │
                         ▼                                   ▼
              ┌──────────────────────┐            ┌──────────────────────┐
              │describeMetadataQuorum│            │describeMetadataQuorum│
              └──────────┬───────────┘            └──────────┬───────────┘
                         │                                   │
           ┌─────────────┴─────────────┐       ┌─────────────┴─────────────┐
           ▼                           ▼       ▼                           ▼
    ┌────────────┐              ┌────────────┐ ┌────────────┐       ┌────────────┐
    │  SUCCESS   │              │ClusterAuth │ │  SUCCESS   │       │ClusterAuth │
    │            │              │ Exception  │ │            │       │ Exception  │
    └─────┬──────┘              └─────┬──────┘ └─────┬──────┘       └─────┬──────┘
          │                           │              │                    │
          ▼                           ▼              ▼                    ▼
    ┌────────────┐              ┌────────────┐ ┌────────────┐       ┌────────────┐
    │Return Data │              │ PROPAGATE  │ │Return Data │       │  Return    │
    │            │              │   ERROR    │ │            │       │  empty()   │
    └────────────┘              │  ════════  │ └────────────┘       │  ════════  │
                                │ Cluster    │                      │ Graceful   │
                                │ shows DOWN │                      │ degradation│
                                └────────────┘                      └────────────┘
```

### WHAT

**File: `StatisticsService.java`**

```java
// Key Fix - loadQuorumInfo() method
.onErrorResume(e -> {
  if (e instanceof UnsupportedVersionException 
      || e instanceof ClusterAuthorizationException) {  // ← NEW
    return Mono.just(Optional.empty());  // Graceful degradation
  }
  return Mono.error(e);
});
```

**File: `ReactiveAdminClient.java`**

```java
// Key Fix - describeCluster() method
// If describeCluster with includeAuthorizedOperations=true fails,
// retry without authorized operations
```

---

## Issue 2: Validation Timeout False Negatives

### WHY

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────┐                                   │
│   │  UI Config Validation Path           │                                   │
│   │  ═════════════════════════════════   │                                   │
│   │                                       │                                   │
│   │  Creates: kui-admin-client-validation-*                                  │
│   │  Lifetime: SHORT (just for validation)                                   │
│   │  Purpose: Check if user config is valid                                  │
│   └─────────────────────────────────────┘                                   │
│                        │                                                     │
│                        ▼                                                     │
│   ┌─────────────────────────────────────┐                                   │
│   │  ORIGINAL TIMEOUT SETTINGS           │                                   │
│   │  ═════════════════════════════════   │                                   │
│   │  request.timeout.ms = 5000 (5s)      │ ◄── TOO TIGHT                    │
│   │  default.api.timeout.ms = 5000 (5s)  │ ◄── FOR CLOUD                    │
│   │  retries = 1                         │ ◄── NO BUFFER                    │
│   └─────────────────────────────────────┘                                   │
│                        │                                                     │
│      Cloud latency     │     Network jitter                                 │
│      (50-200ms RTT)    │     (occasional spikes)                            │
│                        ▼                                                     │
│   ┌─────────────────────────────────────┐                                   │
│   │  ❌ TimeoutException / Disconnect    │                                   │
│   │  ══════════════════════════════════  │                                   │
│   │  UI shows: "Error connecting to      │                                   │
│   │             cluster"                 │                                   │
│   │                                       │                                   │
│   │  Reality: Cluster is perfectly fine! │ ◄── FALSE NEGATIVE               │
│   └─────────────────────────────────────┘                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### HOW

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                        TIMEOUT COMPARISON                                      │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  BEFORE                              AFTER                                     │
│  ══════                              ═════                                     │
│                                                                                │
│  request.timeout.ms = 5000           request.timeout.ms = 30000                │
│  │████████│                          │████████████████████████████████│        │
│  0s      5s                          0s                              30s       │
│                                                                                │
│  default.api.timeout.ms = 5000       default.api.timeout.ms = 30000            │
│  │████████│                          │████████████████████████████████│        │
│  0s      5s                          0s                              30s       │
│                                                                                │
│  retries = 1                         retries = 3                               │
│  │█│                                 │█│█│█│                                   │
│  Attempt 1                           Attempt 1  2  3                           │
│                                                                                │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  WHY 30 SECONDS?                                                               │
│  ════════════════                                                              │
│  • Matches typical cloud SLA expectations                                      │
│  • Accounts for initial connection setup overhead                              │
│  • Allows for TLS handshake + SASL auth negotiation                            │
│  • Provides buffer for cross-region latency                                    │
│                                                                                │
│  WHY 3 RETRIES?                                                                │
│  ═══════════════                                                               │
│  • Transient network blips are common in cloud                                 │
│  • Kafka brokers may be rebalancing                                            │
│  • DNS resolution can occasionally be slow                                     │
│                                                                                │
└───────────────────────────────────────────────────────────────────────────────┘
```

### WHAT

**File: `KafkaServicesValidation.java`**

```java
// Key Fix - validation timeout tuning
Map<String, Object> props = new HashMap<>(bootstrapProperties);
props.put(AdminClientConfig.RETRIES_CONFIG, 3);
props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
props.put(AdminClientConfig.DEFAULT_API_TIMEOUT_MS_CONFIG, 30000);
```

---

## Files Modified

| File | Change | Purpose |
|------|--------|---------|
| `StatisticsService.java` | Catch `ClusterAuthorizationException` in `loadQuorumInfo()` | Treat as "feature unavailable", not "cluster down" |
| `ReactiveAdminClient.java` | Retry `describeCluster()` without authorized operations on auth failure | Handle permission differences gracefully |
| `KafkaServicesValidation.java` | Increase timeouts to 30s, retries to 3 | Make validation tolerant of cloud latency |

---

## Key Insight

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   SAME CREDENTIALS  +  DIFFERENT API  +  DIFFERENT TIMEOUT                  │
│         │                    │                    │                         │
│         └────────────────────┴────────────────────┘                         │
│                              │                                              │
│                              ▼                                              │
│                    DIFFERENT OUTCOMES                                       │
│                                                                             │
│   The fix strategy: Make each path resilient to its specific failure mode  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Debugging Evidence Trail

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WHAT WE OBSERVED                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  OBSERVATION                           CONCLUSION                           │
│  ═══════════                           ══════════                           │
│                                                                             │
│  describeCluster() → ✓ SUCCESS         Credentials work                     │
│                                         │                                   │
│  describeMetadataQuorum() → ✗          API restricted on Confluent Cloud   │
│  ClusterAuthorizationException          │                                   │
│                                         │                                   │
│  Scheduler metrics updating → ✓        Cluster access generally fine       │
│                                         │                                   │
│  Validation client timeout → ✗         Short timeout + cloud latency       │
│  (intermittent)                        = false negatives                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Complete Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           KAFKA UI ADMIN CLIENTS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         ┌─────────────────────┐                             │
│                         │   User Credentials  │                             │
│                         │   (SASL/SSL Config) │                             │
│                         └──────────┬──────────┘                             │
│                                    │                                        │
│              ┌─────────────────────┼─────────────────────┐                  │
│              │                     │                     │                  │
│              ▼                     ▼                     ▼                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│   │  RUNTIME CLIENT  │  │  STATS CLIENT    │  │ VALIDATION CLIENT│         │
│   │  ═══════════════ │  │  ═════════════   │  │ ═════════════════│         │
│   │  Long-lived      │  │  Periodic        │  │  Short-lived     │         │
│   │  Topics/Groups   │  │  Metrics/Quorum  │  │  Config check    │         │
│   └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘         │
│            │                     │                     │                    │
│            │ FIX #1b             │ FIX #1a             │ FIX #2             │
│            ▼                     ▼                     ▼                    │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│   │describeCluster   │  │loadQuorumInfo    │  │ Timeout tuning   │         │
│   │retry without     │  │catch ClusterAuth │  │ 30s + 3 retries  │         │
│   │authorizedOps     │  │return empty()    │  │                  │         │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary Table

| Issue | Symptom | Root Cause | Fix Location | Fix Strategy |
|-------|---------|------------|--------------|--------------|
| **#1a** | Cluster shows DOWN | `describeMetadataQuorum` throws `ClusterAuthorizationException` | `StatisticsService.java` | Catch exception, return `Optional.empty()` |
| **#1b** | `describeCluster` auth errors | `includeAuthorizedOperations=true` requires extra permissions | `ReactiveAdminClient.java` | Retry without authorized operations |
| **#2** | "Error connecting to cluster" during validation | 5s timeout too tight for cloud latency | `KafkaServicesValidation.java` | 30s timeout + 3 retries |

---

## Session Timeline

1. **Initial Symptom**: "Cluster still failing" error in UI
2. **First Discovery**: Unauthorized quorum metadata call causing hard failure in stats path
3. **First Fix**: Graceful degradation (no quorum info, but cluster still healthy)
4. **Second Symptom**: Config validation timeout false negatives appeared
5. **Second Fix**: Increased validation tolerance to match real cloud behavior
6. **Cleanup**: Removed all temporary debug instrumentation, kept only proven fixes

---

## Applicability

This debugging session applies when:

- Connecting Kafka UI to **Confluent Cloud** or other managed Kafka services
- Experiencing intermittent "cluster down" errors despite valid credentials
- Seeing `ClusterAuthorizationException` for quorum-related APIs
- Validation checks failing with timeout errors under normal network conditions

---

## Related Files

- `api/src/main/java/io/kafbat/ui/service/StatisticsService.java`
- `api/src/main/java/io/kafbat/ui/service/ReactiveAdminClient.java`
- `api/src/main/java/io/kafbat/ui/util/KafkaServicesValidation.java`
- `api/src/main/resources/application-dev.yml` (Confluent Cloud config)
