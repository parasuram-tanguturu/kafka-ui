# Kafka UI API Endpoints - Postman Collection Reference

## Overview

This document contains a comprehensive inventory of all **70+ API endpoints** exposed by Kafka UI, organized by domain for use in creating Postman collections or API testing.

**Base URL**: `http://localhost:8080`

---

## Collection Structure

```
┌──────────────────────────────────────────────────────────────────┐
│  📁 Kafka UI API Collection                                      │
├──────────────────────────────────────────────────────────────────┤
│  📁 1. Clusters                                                  │
│  📁 2. Brokers                                                   │
│  📁 3. Topics                                                    │
│  📁 4. Messages                                                  │
│  📁 5. Consumer Groups                                           │
│  📁 6. Schemas (Schema Registry)                                 │
│  📁 7. Kafka Connect                                             │
│  📁 8. KSQL                                                      │
│  📁 9. ACLs                                                      │
│  📁 10. Client Quotas                                            │
│  📁 11. Graphs & Metrics                                         │
│  📁 12. Configuration                                            │
│  📁 13. Authorization                                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Environment Variables

Use these variables in Postman for flexibility across environments:

| Variable | Example Value | Description |
|----------|---------------|-------------|
| `kafbatBaseUrl` | `http://localhost:8080` | API base URL |
| `clusterName` | `local` | Kafka cluster name |
| `topicName` | `test-topic` | Topic name for operations |
| `connectName` | `connect-cluster` | Kafka Connect cluster name |
| `connectorName` | `my-connector` | Connector name |
| `schemaSubject` | `test-value` | Schema Registry subject |
| `consumerId` | `my-consumer-group` | Consumer group ID |
| `brokerId` | `1` | Broker ID |

---

## API Endpoints by Domain

### 1. Clusters (4 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters` | Get all clusters |
| GET | `/api/clusters/{{clusterName}}/metrics` | Get cluster metrics |
| GET | `/api/clusters/{{clusterName}}/stats` | Get cluster stats |
| POST | `/api/clusters/{{clusterName}}/cache` | Update/refresh cluster info |

#### Example Requests

**Get All Clusters**
```http
GET {{kafbatBaseUrl}}/api/clusters
```

**Get Cluster Stats**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/stats
```

---

### 2. Brokers (6 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/brokers` | List all brokers |
| GET | `/api/clusters/{{clusterName}}/brokers/{{brokerId}}/configs` | Get broker config |
| PUT | `/api/clusters/{{clusterName}}/brokers/{{brokerId}}/configs/{{configName}}` | Update broker config |
| GET | `/api/clusters/{{clusterName}}/brokers/{{brokerId}}/metrics` | Get broker metrics |
| GET | `/api/clusters/{{clusterName}}/brokers/logdirs` | Get all broker log dirs |
| PATCH | `/api/clusters/{{clusterName}}/brokers/{{brokerId}}/logdirs` | Update broker log dir |

#### Example Requests

**List Brokers**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/brokers
```

**Get Broker Config**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/brokers/1/configs
```

**Update Broker Config**
```http
PUT {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/brokers/1/configs/log.retention.hours
Content-Type: application/json

{
  "value": "168"
}
```

---

### 3. Topics (16 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/topics` | List topics (paginated) |
| POST | `/api/clusters/{{clusterName}}/topics` | Create topic |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}` | Get topic details |
| PATCH | `/api/clusters/{{clusterName}}/topics/{{topicName}}` | Update topic |
| POST | `/api/clusters/{{clusterName}}/topics/{{topicName}}` | Recreate topic |
| DELETE | `/api/clusters/{{clusterName}}/topics/{{topicName}}` | Delete topic |
| POST | `/api/clusters/{{clusterName}}/topics/{{topicName}}/clone?newTopicName={{newName}}` | Clone topic |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/config` | Get topic configs |
| PATCH | `/api/clusters/{{clusterName}}/topics/{{topicName}}/replications` | Change replication factor |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/partitions` | Get partitions |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/analysis` | Get topic analysis |
| POST | `/api/clusters/{{clusterName}}/topics/{{topicName}}/analysis` | Start topic analysis |
| DELETE | `/api/clusters/{{clusterName}}/topics/{{topicName}}/analysis` | Cancel topic analysis |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/activeproducers` | Get active producers |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/consumer-groups` | Get topic consumer groups |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/connectors` | Get topic connectors |

#### Example Requests

**List Topics (Paginated)**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics?page=1&perPage=25&showInternal=false
```

**Create Topic**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics
Content-Type: application/json

{
  "name": "my-new-topic",
  "partitions": 3,
  "replicationFactor": 1,
  "configs": {
    "retention.ms": "604800000",
    "cleanup.policy": "delete"
  }
}
```

**Get Topic Details**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}
```

**Update Topic Config**
```http
PATCH {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}
Content-Type: application/json

{
  "configs": {
    "retention.ms": "86400000"
  }
}
```

**Delete Topic**
```http
DELETE {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}
```

**Clone Topic**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}/clone?newTopicName=topic-clone
```

**Increase Partitions**
```http
PATCH {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}/partitions
Content-Type: application/json

{
  "totalPartitionsCount": 6
}
```

---

### 4. Messages (7 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/messages` | Get messages (SSE stream) |
| POST | `/api/clusters/{{clusterName}}/topics/{{topicName}}/messages` | Send message |
| DELETE | `/api/clusters/{{clusterName}}/topics/{{topicName}}/messages` | Delete messages |
| GET | `/api/clusters/{{clusterName}}/topics/{{topicName}}/messages/v2` | Get messages v2 |
| GET | `/api/clusters/{{clusterName}}/topic/{{topicName}}/serdes?use=SERIALIZE` | Get serializers |
| POST | `/api/clusters/{{clusterName}}/topics/{{topicName}}/smartfilters` | Register smart filter |
| PUT | `/api/smartfilters/testexecutions` | Test smart filter |

#### Example Requests

**Get Messages**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}/messages?seekType=BEGINNING&limit=100
```

**Send Message**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}/messages
Content-Type: application/json

{
  "key": "my-key",
  "content": "{\"field\": \"value\"}",
  "headers": {
    "header1": "value1"
  },
  "partition": 0
}
```

**Delete Messages (Truncate)**
```http
DELETE {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topics/{{topicName}}/messages?partitions=0&partitions=1
```

**Get Serdes**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/topic/{{topicName}}/serdes?use=DESERIALIZE
```

---

### 5. Consumer Groups (5 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/consumer-groups/paged` | List consumer groups (paged) |
| GET | `/api/clusters/{{clusterName}}/consumer-groups/{{consumerId}}` | Get consumer group details |
| DELETE | `/api/clusters/{{clusterName}}/consumer-groups/{{consumerId}}` | Delete consumer group |
| DELETE | `/api/clusters/{{clusterName}}/consumer-groups/{{consumerId}}/offsets` | Reset offsets |
| DELETE | `/api/clusters/{{clusterName}}/consumer-groups/{{consumerId}}/topics/{{topicName}}` | Delete offsets for topic |

#### Example Requests

**List Consumer Groups**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/consumer-groups/paged?page=1&perPage=25
```

**Get Consumer Group Details**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/consumer-groups/{{consumerId}}
```

**Delete Consumer Group**
```http
DELETE {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/consumer-groups/{{consumerId}}
```

---

### 6. Schemas - Schema Registry (10 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/schemas` | List all schemas |
| POST | `/api/clusters/{{clusterName}}/schemas` | Create new schema |
| GET | `/api/clusters/{{clusterName}}/schemas/{{subject}}` | Get schema subject |
| DELETE | `/api/clusters/{{clusterName}}/schemas/{{subject}}` | Delete schema |
| GET | `/api/clusters/{{clusterName}}/schemas/{{subject}}/versions` | Get schema versions |
| GET | `/api/clusters/{{clusterName}}/schemas/{{subject}}/latest` | Get latest schema |
| GET | `/api/clusters/{{clusterName}}/schemas/{{subject}}/versions/{{version}}` | Get specific version |
| GET | `/api/clusters/{{clusterName}}/schemas/compatibility` | Get global compatibility |
| PUT | `/api/clusters/{{clusterName}}/schemas/{{subject}}/compatibility` | Update schema compatibility |
| POST | `/api/clusters/{{clusterName}}/schemas/{{subject}}/check` | Check schema compatibility |

#### Example Requests

**List Schemas**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/schemas
```

**Create Schema**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/schemas
Content-Type: application/json

{
  "subject": "my-topic-value",
  "schemaType": "AVRO",
  "schema": "{\"type\":\"record\",\"name\":\"MyRecord\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"}]}"
}
```

**Get Schema Versions**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/schemas/{{subject}}/versions
```

**Check Compatibility**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/schemas/{{subject}}/check
Content-Type: application/json

{
  "schemaType": "AVRO",
  "schema": "{\"type\":\"record\",\"name\":\"MyRecord\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"name\",\"type\":\"string\"}]}"
}
```

---

### 7. Kafka Connect (14 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/connects` | List connect clusters |
| GET | `/api/clusters/{{clusterName}}/connectors` | List all connectors |
| GET | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors` | List connectors for cluster |
| POST | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors` | Create connector |
| GET | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}` | Get connector |
| DELETE | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}` | Delete connector |
| POST | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/action/restart` | Restart connector |
| POST | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/action/pause` | Pause connector |
| POST | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/action/resume` | Resume connector |
| GET | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/config` | Get connector config |
| PUT | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/config` | Update connector config |
| GET | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/tasks` | Get connector tasks |
| POST | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/tasks/{{taskId}}/action/restart` | Restart task |
| GET | `/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/offsets` | Get connector offsets |
| GET | `/api/clusters/{{clusterName}}/connects/{{connectName}}/plugins` | List plugins |
| POST | `/api/clusters/{{clusterName}}/connects/{{connectName}}/plugins/{{pluginName}}/config/validate` | Validate plugin config |

#### Example Requests

**List Connect Clusters**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/connects
```

**Create Connector**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors
Content-Type: application/json

{
  "name": "my-file-sink",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "tasks.max": "1",
    "topics": "my-topic",
    "file": "/tmp/output.txt"
  }
}
```

**Restart Connector**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/action/restart
```

**Update Connector Config**
```http
PUT {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/connects/{{connectName}}/connectors/{{connectorName}}/config
Content-Type: application/json

{
  "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
  "tasks.max": "2",
  "topics": "my-topic",
  "file": "/tmp/output.txt"
}
```

---

### 8. KSQL (4 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/clusters/{{clusterName}}/ksql/v2` | Execute KSQL statement |
| GET | `/api/clusters/{{clusterName}}/ksql/tables` | List KSQL tables |
| GET | `/api/clusters/{{clusterName}}/ksql/streams` | List KSQL streams |
| GET | `/api/clusters/{{clusterName}}/ksql/response` | Get KSQL response |

#### Example Requests

**Execute KSQL Statement**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/ksql/v2
Content-Type: application/json

{
  "ksql": "SHOW STREAMS;"
}
```

**List Tables**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/ksql/tables
```

---

### 9. ACLs (7 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/acls` | List ACLs |
| GET | `/api/clusters/{{clusterName}}/acl/csv` | Export ACLs as CSV |
| POST | `/api/clusters/{{clusterName}}/acl` | Create ACL |
| DELETE | `/api/clusters/{{clusterName}}/acl` | Delete ACL |
| POST | `/api/clusters/{{clusterName}}/acl/consumer` | Create consumer ACL |
| POST | `/api/clusters/{{clusterName}}/acl/producer` | Create producer ACL |
| POST | `/api/clusters/{{clusterName}}/acl/streamapp` | Create stream app ACL |

#### Example Requests

**List ACLs**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/acls
```

**Create ACL**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/acl
Content-Type: application/json

{
  "resourceType": "TOPIC",
  "resourceName": "my-topic",
  "resourcePatternType": "LITERAL",
  "principal": "User:alice",
  "host": "*",
  "operation": "READ",
  "permission": "ALLOW"
}
```

**Create Consumer ACL**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/acl/consumer
Content-Type: application/json

{
  "principal": "User:consumer-app",
  "host": "*",
  "consumerGroups": ["my-consumer-group"],
  "topics": ["my-topic"]
}
```

---

### 10. Client Quotas (3 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/clientquotas` | List client quotas |
| POST | `/api/clusters/{{clusterName}}/clientquotas` | Create/update quota |
| DELETE | `/api/clusters/{{clusterName}}/clientquotas` | Delete quota |

#### Example Requests

**List Quotas**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/clientquotas
```

**Create Quota**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/clientquotas
Content-Type: application/json

{
  "entities": [
    {"entityType": "CLIENT_ID", "entityName": "my-producer"}
  ],
  "quotas": {
    "producer_byte_rate": "1048576",
    "consumer_byte_rate": "2097152"
  }
}
```

---

### 11. Graphs & Metrics (4 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/clusters/{{clusterName}}/graphs/descriptions` | Get available graphs |
| POST | `/api/clusters/{{clusterName}}/graphs/prometheus` | Get graph data |
| GET | `/metrics` | Expose all metrics (Prometheus format) |
| GET | `/metrics/{{clusterName}}` | Expose cluster metrics |

#### Example Requests

**Get Graph Descriptions**
```http
GET {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/graphs/descriptions
```

**Get Graph Data**
```http
POST {{kafbatBaseUrl}}/api/clusters/{{clusterName}}/graphs/prometheus
Content-Type: application/json

{
  "query": "kafka_server_BrokerTopicMetrics_MessagesInPerSec",
  "start": "2024-01-01T00:00:00Z",
  "end": "2024-01-01T01:00:00Z",
  "step": "60"
}
```

**Get Prometheus Metrics**
```http
GET {{kafbatBaseUrl}}/metrics
```

---

### 12. Configuration (6 endpoints)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/info` | Get application info |
| GET | `/api/config` | Get current application config |
| PUT | `/api/config` | Apply new config & restart app |
| PUT | `/api/config/validated` | Validate config before applying |
| POST | `/api/config/relatedfiles` | Upload SSL certs/config files |
| GET | `/api/config/authentication` | Get authentication config |

#### Example Requests

**Get App Info**
```http
GET {{kafbatBaseUrl}}/api/info
```

**Get Current Config**
```http
GET {{kafbatBaseUrl}}/api/config
```

**Apply Config & Restart (Dynamic Cluster Add)**
```http
PUT {{kafbatBaseUrl}}/api/config
Content-Type: application/json

{
  "config": {
    "properties": {
      "kafka": {
        "clusters": [
          {
            "name": "local",
            "bootstrapServers": "localhost:9092"
          },
          {
            "name": "confluent-cloud",
            "bootstrapServers": "pkc-xxxxx.region.gcp.confluent.cloud:9092",
            "properties": {
              "security.protocol": "SASL_SSL",
              "sasl.mechanism": "PLAIN",
              "sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username='API_KEY' password='API_SECRET';"
            }
          }
        ]
      }
    }
  }
}
```

**Validate Config**
```http
PUT {{kafbatBaseUrl}}/api/config/validated
Content-Type: application/json

{
  "properties": {
    "kafka": {
      "clusters": [
        {
          "name": "test-cluster",
          "bootstrapServers": "localhost:9092"
        }
      ]
    }
  }
}
```

**Upload Config File (SSL cert, truststore)**
```http
POST {{kafbatBaseUrl}}/api/config/relatedfiles
Content-Type: multipart/form-data

[file attachment]
```

---

### 13. Authorization (1 endpoint)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/authorization` | Get user permissions |

#### Example Request

**Get User Permissions**
```http
GET {{kafbatBaseUrl}}/api/authorization
```

---

## Testing Workflow

### Recommended Test Order

1. **Health Check**: `GET /api/clusters` - Verify API connectivity
2. **Cluster Info**: `GET /api/clusters/{clusterName}/stats` - Verify cluster access
3. **Topics**: Create, list, get details, delete
4. **Messages**: Send and consume messages
5. **Consumer Groups**: View and manage
6. **Schema Registry**: If configured
7. **Kafka Connect**: If configured
8. **ACLs**: If enabled

### Common Response Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request (validation error) |
| 401 | Unauthorized |
| 403 | Forbidden (RBAC) |
| 404 | Not Found |
| 500 | Internal Server Error |

---

## References

- **OpenAPI Spec**: `contract/src/main/resources/swagger/kafbat-ui-api.yaml`
- **Controllers**: `api/src/main/java/io/kafbat/ui/controller/`
- **Local Dev**: See `LOCAL_DEVELOPMENT.md` for running the API locally
