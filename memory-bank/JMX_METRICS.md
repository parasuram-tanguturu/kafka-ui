# JMX Metrics in Kafbat UI

> Understanding Java Management Extensions (JMX) and how Kafbat UI uses it to collect Kafka metrics.

---

## What is JMX?

**JMX (Java Management Extensions)** is a Java technology for monitoring and managing Java applications at runtime. It provides a standard way to expose operational data (metrics, configuration) from any Java application.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           JMX ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐         ┌─────────────────┐                          │
│   │  Kafka Broker   │         │  Kafbat UI      │                          │
│   │                 │         │                 │                          │
│   │  ┌───────────┐  │  JMX    │  ┌───────────┐  │                          │
│   │  │ MBeans    │◀─┼─────────┼──│ JMX Client│  │                          │
│   │  │ (metrics) │  │  :9997  │  │           │  │                          │
│   │  └───────────┘  │         │  └───────────┘  │                          │
│   │                 │         │                 │                          │
│   └─────────────────┘         └─────────────────┘                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **MBean** | Managed Bean - a Java object that represents a manageable resource |
| **MBean Server** | Registry that holds all MBeans in a JVM |
| **JMX Connector** | Protocol for remote access (usually RMI over TCP) |
| **JMX Port** | Network port for JMX connections (commonly 9997 for Kafka) |

---

## What Metrics Does JMX Expose?

Kafka brokers expose hundreds of metrics via JMX. Kafbat UI collects:

### Broker Metrics

| Metric | Description |
|--------|-------------|
| `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec` | Messages received per second |
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | Bytes received per second |
| `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec` | Bytes sent per second |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Partitions with insufficient replicas |

### Consumer Group Metrics

| Metric | Description |
|--------|-------------|
| `kafka.server:type=FetcherLagMetrics,name=ConsumerLag` | Consumer lag (messages behind) |
| `kafka.consumer:type=consumer-fetch-manager-metrics` | Fetch rate, latency |

### Topic/Partition Metrics

| Metric | Description |
|--------|-------------|
| `kafka.log:type=Log,name=Size` | Log size per partition |
| `kafka.log:type=Log,name=NumLogSegments` | Number of log segments |

---

## JMX Configuration in Kafbat UI

### Local Development (`application-local.yml`)

```yaml
kafka:
  clusters:
    - name: local
      bootstrapServers: localhost:9092
      metrics:
        port: 9997      # JMX port
        type: JMX       # Enable JMX metrics collection
```

### Docker Kafka Configuration (`kafbat-ui.yaml`)

```yaml
kafka0:
  ports:
    - "9092:9092"     # Kafka protocol
    - "9997:9997"     # JMX port
  environment:
    KAFKA_JMX_PORT: 9997
    KAFKA_JMX_HOSTNAME: localhost
    KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote 
                     -Dcom.sun.management.jmxremote.authenticate=false 
                     -Dcom.sun.management.jmxremote.ssl=false"
```

---

## JMX Connection Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        JMX CONNECTION FLOW                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Kafbat UI startup                                                      │
│      │                                                                      │
│      │ Read config: metrics.port=9997, metrics.type=JMX                    │
│      ▼                                                                      │
│   2. Create JMX connection URL                                              │
│      │                                                                      │
│      │ service:jmx:rmi:///jndi/rmi://localhost:9997/jmxrmi                 │
│      ▼                                                                      │
│   3. Connect to Kafka's MBean Server                                        │
│      │                                                                      │
│      │ JMXConnector.connect()                                              │
│      ▼                                                                      │
│   4. Query MBeans periodically                                              │
│      │                                                                      │
│      │ getMBeanServerConnection().queryMBeans(...)                         │
│      ▼                                                                      │
│   5. Display metrics in UI                                                  │
│      │                                                                      │
│      └── Dashboard → Broker metrics, Topic metrics, etc.                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Common JMX Errors

### 1. SSL JMX Error (Java 9+)

```
SSL can't be enabled for JMX retrieval. Make sure your java app is running with 
'--add-opens java.rmi/javax.rmi.ssl=ALL-UNNAMED' arg
```

**WHY**: Java 9+ module system restricts reflective access to internal JMX SSL classes.

**IMPACT**: SSL JMX won't work, but plain JMX works fine.

**FIX** (if you need SSL JMX):

```bash
./gradlew :api:bootRun \
  --args='--spring.profiles.active=local' \
  -Dorg.gradle.jvmargs='--add-opens java.rmi/javax.rmi.ssl=ALL-UNNAMED'
```

**For local development**: Not needed. Docker Kafka uses plain JMX.

---

### 2. Connection Refused

```
Connection refused: localhost:9997
```

**WHY**: Kafka broker not running or JMX not enabled.

**FIX**:

```bash
# Ensure Kafka container is running
docker ps | grep kafka0

# Verify JMX port is exposed
docker port kafka0
# Should show: 9997/tcp -> 0.0.0.0:9997
```

---

### 3. Authentication Required

```
Authentication failed! Credentials required
```

**WHY**: Kafka broker has JMX authentication enabled.

**FIX**: Add credentials to config:

```yaml
kafka:
  clusters:
    - name: production
      metrics:
        port: 9997
        type: JMX
        username: jmxuser
        password: jmxpass
```

---

## JMX vs Other Metrics Sources

Kafbat UI supports multiple metrics sources:

| Source | Protocol | Use Case |
|--------|----------|----------|
| **JMX** | RMI/TCP | Direct access to Kafka JVM metrics |
| **Prometheus** | HTTP | Kafka with JMX Exporter sidecar |
| **AWS MSK** | AWS API | Amazon Managed Streaming for Kafka |

### Prometheus Alternative

If JMX is problematic, use Prometheus with JMX Exporter:

```yaml
kafka:
  clusters:
    - name: local
      metrics:
        type: PROMETHEUS
        port: 11001  # JMX Exporter port
```

---

## Verifying JMX Works

### Check in Logs

Look for successful JMX collection:

```
DEBUG i.k.u.s.m.s.jmx.JmxMetricsRetriever : Collecting JMX metrics for 
service:jmx:rmi:///jndi/rmi://localhost:9997/jmxrmi
```

### Check in UI

1. Open http://localhost:3000
2. Go to **Dashboard** or **Brokers**
3. Verify metrics are displayed:
   - Messages In/Out per second
   - Bytes In/Out per second
   - Active controllers
   - Under-replicated partitions

### Test JMX Manually

```bash
# Using jconsole (GUI)
jconsole localhost:9997

# Using cmdline-jmxclient
java -jar cmdline-jmxclient.jar - localhost:9997 kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
```

---

## Security Considerations

### Local Development

```yaml
# No authentication (fine for local)
KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote.authenticate=false 
                 -Dcom.sun.management.jmxremote.ssl=false"
```

### Production

```yaml
# Enable authentication and SSL
KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote.authenticate=true 
                 -Dcom.sun.management.jmxremote.ssl=true
                 -Dcom.sun.management.jmxremote.password.file=/path/to/jmxremote.password
                 -Dcom.sun.management.jmxremote.access.file=/path/to/jmxremote.access"
```

---

## Related Documentation

- [Oracle JMX Guide](https://docs.oracle.com/javase/tutorial/jmx/)
- [Kafka Monitoring Docs](https://kafka.apache.org/documentation/#monitoring)
- [Kafbat UI Metrics Configuration](https://ui.docs.kafbat.io/configuration/metrics)
