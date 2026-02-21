# Kafka Quorum Explained (KRaft)

> A beginner-friendly explanation of quorum, consensus, and the `describeMetadataQuorum` API in Apache Kafka.

---

## Table of Contents

- [WHY: The Problem Quorum Solves](#why-the-problem-quorum-solves)
- [HOW: Quorum Works](#how-quorum-works)
- [Kafka's Implementation: KRaft](#kafkas-implementation-kraft)
- [The describeMetadataQuorum API](#the-describemetadataquorum-api)
- [Why Confluent Cloud Blocks This API](#why-confluent-cloud-blocks-this-api)
- [Summary](#summary)

---

## WHY: The Problem Quorum Solves

### The Core Problem: Who's in Charge?

Imagine you have 3 Kafka brokers. They all need to agree on things like:
- "Which broker is the leader for partition X?"
- "What topics exist?"
- "What's the current cluster configuration?"

**What happens if they disagree?**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        THE SPLIT-BRAIN PROBLEM                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Network partition occurs...                                               │
│                                                                             │
│   ┌─────────┐         ╳ ╳ ╳         ┌─────────┐                            │
│   │ Broker 1│ ◄───── network ─────► │ Broker 2│                            │
│   │         │         down          │         │                            │
│   └─────────┘                       └─────────┘                            │
│        │                                 │                                  │
│        ▼                                 ▼                                  │
│   "I'm the leader!"              "I'm the leader!"                         │
│                                                                             │
│   🔥 CHAOS: Two leaders, data corruption, inconsistency 🔥                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Solution: Quorum (Voting)

**Quorum** = "A minimum number of members that must agree before a decision is valid"

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REAL-WORLD ANALOGY                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Think of a board meeting with 5 directors:                                │
│                                                                             │
│   • To pass a resolution, you need a MAJORITY (3+ votes)                    │
│   • If the room splits into two groups (2 vs 3), only the group             │
│     with 3+ can make valid decisions                                        │
│   • The minority group KNOWS it can't act alone                             │
│                                                                             │
│   This prevents two groups from making conflicting decisions!               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## HOW: Quorum Works

### The Math: Why Majority Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        QUORUM MATH                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Total Nodes: N                                                            │
│   Quorum Size: (N/2) + 1  (majority)                                        │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────┐           │
│   │  N = 3 nodes  →  Quorum = 2  →  Can tolerate 1 failure      │           │
│   │  N = 5 nodes  →  Quorum = 3  →  Can tolerate 2 failures     │           │
│   │  N = 7 nodes  →  Quorum = 4  →  Can tolerate 3 failures     │           │
│   └─────────────────────────────────────────────────────────────┘           │
│                                                                             │
│   KEY INSIGHT: You can NEVER have two majorities at the same time!          │
│   ═══════════                                                               │
│   If 5 nodes split into 2 and 3, only the group of 3 has quorum.            │
│   The group of 2 MUST wait (or join the other group).                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Visual: Quorum Tolerance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FAULT TOLERANCE VS CLUSTER SIZE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Nodes    Quorum    Failures Tolerated    Common Use Case                  │
│   ─────    ──────    ──────────────────    ───────────────                  │
│     1        1             0                Dev/Test                        │
│     3        2             1                Small production                │
│     5        3             2                Standard production             │
│     7        4             3                High availability               │
│                                                                             │
│   Formula: Failures Tolerated = (N - 1) / 2                                 │
│                                                                             │
│   WHY ODD NUMBERS?                                                          │
│   ════════════════                                                          │
│   • 3 nodes: quorum = 2, tolerate 1 failure                                 │
│   • 4 nodes: quorum = 3, tolerate 1 failure (same as 3 nodes!)              │
│   • Adding a 4th node doesn't improve fault tolerance                       │
│   • Odd numbers are more efficient                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Kafka's Implementation: KRaft

### Old Way: ZooKeeper (External Coordinator)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OLD ARCHITECTURE (ZooKeeper)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│            ┌───────────────────────────────────┐                            │
│            │         ZooKeeper Cluster         │ ◄── External system        │
│            │  (manages metadata & elections)   │     (separate process)     │
│            └─────────────────┬─────────────────┘                            │
│                              │                                              │
│         ┌────────────────────┼────────────────────┐                         │
│         ▼                    ▼                    ▼                         │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐                   │
│   │ Broker 1 │         │ Broker 2 │         │ Broker 3 │                   │
│   └──────────┘         └──────────┘         └──────────┘                   │
│                                                                             │
│   PROBLEMS:                                                                 │
│   • Extra infrastructure to manage                                          │
│   • Network hops between Kafka and ZooKeeper                                │
│   • Complexity in operations                                                │
│   • ZooKeeper scaling limitations                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### New Way: KRaft (Kafka Raft)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        NEW ARCHITECTURE (KRaft)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   KRaft = Kafka Raft (Kafka's implementation of the Raft consensus algo)    │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                    CONTROLLER QUORUM                             │       │
│   │  ┌────────────┐   ┌────────────┐   ┌────────────┐               │       │
│   │  │Controller 1│   │Controller 2│   │Controller 3│               │       │
│   │  │  (LEADER)  │◄──│ (FOLLOWER) │   │ (FOLLOWER) │               │       │
│   │  └────────────┘   └────────────┘   └────────────┘               │       │
│   │        │                │                │                       │       │
│   │        └────────────────┴────────────────┘                       │       │
│   │                         │                                        │       │
│   │              Replicate metadata log                              │       │
│   └─────────────────────────┬───────────────────────────────────────┘       │
│                             │                                               │
│         ┌───────────────────┼───────────────────┐                          │
│         ▼                   ▼                   ▼                          │
│   ┌──────────┐        ┌──────────┐        ┌──────────┐                     │
│   │ Broker 1 │        │ Broker 2 │        │ Broker 3 │                     │
│   │ (data)   │        │ (data)   │        │ (data)   │                     │
│   └──────────┘        └──────────┘        └──────────┘                     │
│                                                                             │
│   Controllers can be:                                                       │
│   • Combined mode: Controller + Broker on same node                         │
│   • Dedicated mode: Separate controller and broker nodes                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What the Controller Quorum Manages

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    QUORUM RESPONSIBILITIES                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   The Controller Quorum is the "brain" of the cluster:                      │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────┐           │
│   │  CLUSTER METADATA                                            │           │
│   │  ════════════════                                            │           │
│   │  • Topic definitions (name, partitions, replication factor) │           │
│   │  • Partition assignments (which broker owns what)            │           │
│   │  • Broker registration (which brokers are alive)             │           │
│   │  • Configuration changes (topic/broker configs)              │           │
│   └─────────────────────────────────────────────────────────────┘           │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────┐           │
│   │  COORDINATION                                                │           │
│   │  ════════════                                                │           │
│   │  • Leader elections (when a partition leader fails)         │           │
│   │  • ISR (In-Sync Replica) management                         │           │
│   │  • Access control lists (ACLs)                               │           │
│   │  • Quotas and rate limiting                                  │           │
│   └─────────────────────────────────────────────────────────────┘           │
│                                                                             │
│   All stored in a REPLICATED LOG (similar to a Kafka topic itself!)        │
│   The quorum ensures this log is consistent across all controllers.         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The describeMetadataQuorum API

### What It Returns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    describeMetadataQuorum() OUTPUT                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   {                                                                         │
│     "leaderId": 1,           // Which controller is the leader             │
│     "leaderEpoch": 42,       // How many leader elections have happened    │
│     "highWatermark": 12345,  // Latest committed offset in metadata log    │
│                                                                             │
│     "voters": [              // Controllers that can vote (full members)   │
│       {                                                                     │
│         "replicaId": 1,                                                     │
│         "logEndOffset": 12345,    // How far this replica has caught up    │
│         "lastFetchTimestamp": ..., // Last time it fetched from leader     │
│         "lastCaughtUpTimestamp": ...                                        │
│       },                                                                    │
│       { "replicaId": 2, ... },                                              │
│       { "replicaId": 3, ... }                                               │
│     ],                                                                      │
│                                                                             │
│     "observers": [           // Non-voting replicas (read-only followers)  │
│       { ... }                                                               │
│     ]                                                                       │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Visual: Metadata Log Replication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    QUORUM HEALTH VISUALIZATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Metadata Log:  [0]──[1]──[2]──...──[12340]──[12341]──[12342]──[12345]    │
│                                                         │        │    │     │
│                                                         │        │    │     │
│   Controller 1 (LEADER) ────────────────────────────────┴────────┴────┘     │
│   logEndOffset: 12345  ✓ (fully caught up - it's the leader!)              │
│                                                                             │
│   Controller 2 (FOLLOWER) ──────────────────────────────┴────────┴──┘       │
│   logEndOffset: 12343  ⚠ (2 entries behind)                                │
│                                                                             │
│   Controller 3 (FOLLOWER) ──────────────────────────────┴──┘                │
│   logEndOffset: 12341  ⚠ (4 entries behind)                                │
│                                                                             │
│   HIGH WATERMARK: 12341                                                     │
│   (Entries up to here are COMMITTED - majority has them)                    │
│                                                                             │
│   WHY HIGH WATERMARK?                                                       │
│   ════════════════════                                                      │
│   • Leader has written up to 12345                                          │
│   • But only 12341 is replicated to majority (2 of 3)                       │
│   • 12341 is "committed" - safe to acknowledge to clients                   │
│   • 12342-12345 are "uncommitted" - could be lost if leader fails           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Use Cases for describeMetadataQuorum

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHEN IS THIS API USEFUL?                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   MONITORING & ALERTING                                                     │
│   ═════════════════════                                                     │
│   • Check if all controllers are healthy                                    │
│   • Detect replication lag (followers falling behind)                       │
│   • Monitor leader election frequency (epoch changes)                       │
│                                                                             │
│   OPERATIONS                                                                │
│   ══════════                                                                │
│   • Verify quorum is stable before maintenance                              │
│   • Debug slow metadata propagation                                         │
│   • Identify unhealthy controllers                                          │
│                                                                             │
│   EXAMPLE ALERTS:                                                           │
│   ═══════════════                                                           │
│   • "Controller 3 is 1000+ entries behind leader"                           │
│   • "Leader epoch increased 5 times in last hour"                           │
│   • "Only 2 of 3 controllers responding"                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Why Confluent Cloud Blocks This API

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONFLUENT CLOUD RESTRICTION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Confluent Cloud is a MANAGED service:                                     │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────┐           │
│   │                    CONFLUENT MANAGES                         │           │
│   │   ┌───────────────────────────────────────────────────┐     │           │
│   │   │  • Controller quorum (you never see it)            │     │           │
│   │   │  • Infrastructure (VMs, networking, storage)       │     │           │
│   │   │  • Replication (automatic, multi-AZ)               │     │           │
│   │   │  • Leader elections (handled transparently)        │     │           │
│   │   │  • Upgrades and patches                            │     │           │
│   │   └───────────────────────────────────────────────────┘     │           │
│   └─────────────────────────────────────────────────────────────┘           │
│                              │                                              │
│                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────┐           │
│   │                      YOU MANAGE                              │           │
│   │   ┌───────────────────────────────────────────────────┐     │           │
│   │   │  • Topics (create, configure, delete)              │     │           │
│   │   │  • Consumers and consumer groups                   │     │           │
│   │   │  • Producers and message publishing                │     │           │
│   │   │  • Schema Registry (schemas, compatibility)        │     │           │
│   │   │  • Connectors (Kafka Connect)                      │     │           │
│   │   │  • Your application code                           │     │           │
│   │   └───────────────────────────────────────────────────┘     │           │
│   └─────────────────────────────────────────────────────────────┘           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why the API is Blocked

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REASONS FOR RESTRICTION                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. NOT YOUR CONCERN                                                       │
│      • Quorum health is Confluent's operational responsibility              │
│      • You pay them to manage it - that's the point of managed service     │
│                                                                             │
│   2. SECURITY                                                               │
│      • Exposing internal topology could leak infrastructure details         │
│      • Controller IDs, replication lag = potential attack surface info     │
│                                                                             │
│   3. PERMISSIONS                                                            │
│      • API requires CLUSTER-level DESCRIBE permission                       │
│      • Managed Kafka doesn't grant cluster-admin to tenants                 │
│                                                                             │
│   4. IRRELEVANT FOR USE CASE                                                │
│      • For producing/consuming messages, you don't need quorum info         │
│      • Confluent provides their own monitoring dashboards                   │
│                                                                             │
│   RESULT: ClusterAuthorizationException                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Self-Managed vs Managed: API Availability

| API | Self-Managed Kafka | Confluent Cloud |
|-----|-------------------|-----------------|
| `describeCluster` | ✅ Full access | ✅ Works |
| `listTopics` | ✅ Full access | ✅ Works |
| `describeTopics` | ✅ Full access | ✅ Works |
| `createTopics` | ✅ Full access | ✅ Works |
| `describeMetadataQuorum` | ✅ Full access | ❌ `ClusterAuthorizationException` |
| `describeFeatures` | ✅ Full access | ⚠️ Limited |
| `alterConfigs` (cluster) | ✅ Full access | ❌ Not allowed |

---

## Summary

### Key Terms

| Term | Meaning |
|------|---------|
| **Quorum** | Minimum number of nodes that must agree for a decision to be valid (usually majority) |
| **Consensus** | The process of getting distributed nodes to agree on a value |
| **Raft** | A consensus algorithm (simpler than Paxos) - basis for KRaft |
| **KRaft** | Kafka Raft - Kafka's built-in consensus protocol (replaces ZooKeeper) |
| **Controller** | Special Kafka node that participates in metadata quorum |
| **Leader** | The controller currently coordinating metadata changes |
| **Follower** | Controllers that replicate metadata from the leader |
| **Voter** | Controller that participates in leader elections |
| **Observer** | Non-voting replica (for read scaling) |
| **Epoch** | Counter that increments on each leader election |
| **High Watermark** | Latest offset that a majority of voters have acknowledged |
| `describeMetadataQuorum` | Admin API to inspect quorum health and replication lag |

### Visual Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QUORUM IN ONE PICTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        ┌─────────────────┐                                  │
│                        │   METADATA LOG  │                                  │
│                        │   (replicated)  │                                  │
│                        └────────┬────────┘                                  │
│                                 │                                           │
│              ┌──────────────────┼──────────────────┐                        │
│              ▼                  ▼                  ▼                        │
│        ┌──────────┐      ┌──────────┐      ┌──────────┐                    │
│        │Controller│      │Controller│      │Controller│                    │
│        │    1     │      │    2     │      │    3     │                    │
│        │ (LEADER) │◄─────│(FOLLOWER)│      │(FOLLOWER)│                    │
│        └──────────┘      └──────────┘      └──────────┘                    │
│              │                                                              │
│              │           QUORUM RULES:                                      │
│              │           • 3 controllers → need 2 to agree                  │
│              │           • Can survive 1 controller failure                 │
│              │           • No split-brain possible                          │
│              │                                                              │
│              ▼                                                              │
│        ┌──────────────────────────────────────────┐                        │
│        │              DATA BROKERS                │                        │
│        │  (store actual messages, serve clients)  │                        │
│        └──────────────────────────────────────────┘                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Related Reading

- [KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum)
- [Apache Kafka KRaft Documentation](https://kafka.apache.org/documentation/#kraft)
- [The Raft Consensus Algorithm](https://raft.github.io/)
- [Confluent Cloud Architecture](https://docs.confluent.io/cloud/current/overview.html)

---

## Relevance to Kafka UI

When Kafka UI connects to a cluster, it tries to gather as much information as possible, including quorum info. On managed clusters like Confluent Cloud:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   In SELF-MANAGED Kafka: describeMetadataQuorum() → useful for monitoring  │
│                                                                             │
│   In CONFLUENT CLOUD:    describeMetadataQuorum() → ClusterAuthorizationException
│                          (Confluent manages quorum, you don't need to see it)
│                                                                             │
│   KAFKA UI FIX: Treat this exception as "feature not available" rather     │
│                 than "cluster is broken"                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

See: `CONFLUENT_CLOUD_DEBUG_SESSION.md` for the related debugging session.
