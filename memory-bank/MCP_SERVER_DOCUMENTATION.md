# Kafka UI MCP Server Documentation

## Table of Contents

- [What is MCP?](#what-is-mcp)
- [Architecture Overview](#architecture-overview)
- [How to Enable MCP](#how-to-enable-mcp)
  - [Configuration Property](#configuration-property)
  - [Environment Variable](#environment-variable)
  - [Docker Compose](#docker-compose)
  - [Running Locally (Gradle)](#running-locally-gradle)
- [Code Evidence](#code-evidence)
  - [MCP Configuration](#1-mcp-configuration-mcpconfigjava)
  - [McpTool Marker Interface](#2-mcptool-marker-interface)
  - [Tool Specification Generator](#3-tool-specification-generator)
  - [Controller Implementation](#4-controller-implementation-example)
- [Available MCP Tools](#available-mcp-tools)
- [Transport Layer](#transport-layer)
- [Dependencies](#dependencies)
- [Usage Examples](#usage-examples)
- [Configuring Cursor IDE to Use Kafka UI MCP](#configuring-cursor-ide-to-use-kafka-ui-mcp)
  - [Prerequisites](#prerequisites)
  - [Step 1: Open Cursor Settings](#step-1-open-cursor-settings)
  - [Step 2: Add MCP Server Configuration](#step-2-add-mcp-server-configuration)
  - [Configuration for Different Environments](#configuration-for-different-environments)
  - [Troubleshooting](#troubleshooting)
- [Server Capabilities](#server-capabilities)
- [File References](#file-references)
- [External Documentation](#external-documentation)

---

## What is MCP?

**Model Context Protocol (MCP)** is an open protocol that enables AI assistants and LLM-based tools to interact with external systems. Kafka UI implements an MCP Server, allowing AI agents to manage Kafka clusters programmatically.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MCP in Action                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────┐                      ┌───────────────────────────┐  │
│   │   AI Agent    │                      │      Kafka Cluster        │  │
│   │  (Claude,     │ ─────── MCP ───────▶ │  ┌─────┐ ┌─────┐ ┌─────┐  │  │
│   │   Cursor,     │                      │  │Topic│ │Topic│ │Topic│  │  │
│   │   etc.)       │ ◀────── SSE ──────── │  └─────┘ └─────┘ └─────┘  │  │
│   └───────────────┘                      └───────────────────────────┘  │
│                                                                         │
│   "List all topics"  ──▶  getTopics({clusterName: "prod"})             │
│   "Create topic X"   ──▶  createTopic({clusterName: "prod", ...})      │
│   "Show consumers"   ──▶  getConsumerGroups({clusterName: "prod"})     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Overview

### High-Level Flow

```
                                    KAFKA UI MCP SERVER
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ┌────────────┐      ┌─────────────────────┐      ┌────────────────────┐    │
│  │            │      │    SSE Transport    │      │                    │    │
│  │ MCP Client │─────▶│  /mcp/message (IN)  │─────▶│  McpAsyncServer    │    │
│  │            │◀─────│  /mcp/sse (OUT)     │◀─────│                    │    │
│  └────────────┘      └─────────────────────┘      └─────────┬──────────┘    │
│                                                             │               │
│                                                             ▼               │
│                                          ┌──────────────────────────────┐   │
│                                          │  McpSpecificationGenerator   │   │
│                                          │  ─────────────────────────── │   │
│                                          │  • Scans McpTool controllers │   │
│                                          │  • Reads @Operation metadata │   │
│                                          │  • Generates JSON schemas    │   │
│                                          └──────────────┬───────────────┘   │
│                                                         │                   │
│     ┌───────────────────────────────────────────────────┼───────────────┐   │
│     │                                                   │               │   │
│     ▼                   ▼                   ▼           ▼               │   │
│ ┌─────────┐       ┌──────────┐       ┌───────────┐ ┌──────────┐        │   │
│ │ Topics  │       │ Brokers  │       │ Consumer  │ │ Schemas  │  ...   │   │
│ │Controller│       │Controller│       │  Groups   │ │Controller│        │   │
│ └─────────┘       └──────────┘       └───────────┘ └──────────┘        │   │
│                                                                         │   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                        ┌───────────────────────────┐
                        │      Kafka Cluster        │
                        │  (via ReactiveAdminClient)│
                        └───────────────────────────┘
```

---

## How to Enable MCP

### Configuration Property

```properties
# application.properties or application.yml
mcp.enabled=true
```

### Environment Variable

```bash
export MCP_ENABLED=true
```

### Docker Compose

```yaml
services:
  kafka-ui:
    image: ghcr.io/kafbat/kafka-ui:latest
    environment:
      MCP_ENABLED: "true"
```

### Running Locally (Gradle)

**Prerequisites:** Java 25 via SDKMAN

```bash
# Set Java version (required)
sdk use java 25.0.2-zulu

# Run with local profile (pre-configured Kafka cluster)
./gradlew :api:bootRun --args='--spring.profiles.active=local'

# Or run with dev profile (dynamic cluster management via UI)
./gradlew :api:bootRun --args='--spring.profiles.active=dev'
```

---

## Code Evidence

### 1. MCP Configuration (`McpConfig.java`)

The MCP server is conditionally enabled via Spring's `@ConditionalOnProperty`:

```java
@Configuration
@RequiredArgsConstructor
@ConditionalOnProperty(value = "mcp.enabled", havingValue = "true")
public class McpConfig {

  private final List<McpTool> mcpTools;
  private final McpSpecificationGenerator mcpSpecificationGenerator;

  // SSE transport endpoints
  @Bean
  public WebFluxSseServerTransportProvider sseServerTransport(ObjectMapper mapper) {
    return new WebFluxSseServerTransportProvider(mapper, "/mcp/message", "/mcp/sse");
  }

  @Bean
  public McpAsyncServer mcpServer(WebFluxSseServerTransportProvider transport) {
    var capabilities = McpSchema.ServerCapabilities.builder()
        .resources(false, true)
        .tools(true)      // Tools support enabled
        .prompts(false)   // Prompts not supported
        .logging()        // Logging enabled
        .build();

    return McpServer.async(transport)
        .serverInfo("Kafka UI MCP", "0.0.1")
        .capabilities(capabilities)
        .tools(tools())   // Register all tools
        .build();
  }
}
```

**Source:** `api/src/main/java/io/kafbat/ui/config/McpConfig.java`

### 2. McpTool Marker Interface

Controllers that expose MCP tools implement this marker interface:

```java
package io.kafbat.ui.service.mcp;

public interface McpTool {
}
```

**Source:** `api/src/main/java/io/kafbat/ui/service/mcp/McpTool.java`

### 3. Tool Specification Generator

The `McpSpecificationGenerator` automatically converts controller methods to MCP tools:

```java
@Component
@RequiredArgsConstructor
public class McpSpecificationGenerator {
  
  public List<AsyncToolSpecification> convertTool(McpTool controller) {
    List<AsyncToolSpecification> result = new ArrayList<>();
    Class<?> targetClass = AopUtils.getTargetClass(controller);
    
    for (Method method : targetClass.getMethods()) {
      // Skip deprecated methods
      Deprecated deprecated = AnnotationUtils.findAnnotation(method, Deprecated.class);
      if (deprecated == null) {
        // Find methods with @Operation annotation
        Operation annotation = AnnotationUtils.findAnnotation(method, Operation.class);
        if (annotation != null) {
          result.add(this.convertOperation(method, annotation, controller));
        }
      }
    }
    return result;
  }
}
```

**Key Features:**
- Uses Swagger `@Operation` annotations for tool metadata
- Generates JSON Schema from Java types automatically
- Handles `Mono<T>`, `Flux<T>`, and `ResponseEntity<T>` return types
- Skips `@Deprecated` methods

**Source:** `api/src/main/java/io/kafbat/ui/service/mcp/McpSpecificationGenerator.java`

### 4. Controller Implementation Example

Controllers expose their REST endpoints as MCP tools by implementing `McpTool`:

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class TopicsController extends AbstractController implements TopicsApi, McpTool {
  // All @Operation annotated methods become MCP tools
}
```

---

## Available MCP Tools

### Controllers Exposing MCP Tools

| Controller | Description | Example Tools |
|------------|-------------|---------------|
| `TopicsController` | Topic management | `getTopics`, `createTopic`, `deleteTopic`, `updateTopic`, `cloneTopic`, `recreateTopic`, `getTopicConfigs` |
| `BrokersController` | Broker operations | `getBrokers`, `getBroker`, `updateBrokerConfig` |
| `ConsumerGroupsController` | Consumer group management | `getConsumerGroups`, `getConsumerGroup`, `deleteConsumerGroup`, `resetOffsets` |
| `MessagesController` | Message operations | `getTopicMessages`, `sendMessage` |
| `SchemasController` | Schema Registry | `getSchemas`, `createSchema`, `deleteSchema`, `getLatestSchema` |
| `KafkaConnectController` | Kafka Connect | `getConnectors`, `createConnector`, `deleteConnector`, `restartConnector` |
| `KsqlController` | ksqlDB queries | `executeKsqlQuery` |
| `AclsController` | ACL management | `listAcls`, `createAcl`, `deleteAcl` |
| `ClientQuotasController` | Client quotas | `getQuotas`, `upsertQuota` |
| `ClustersController` | Cluster info | `getClusters`, `getClusterStats` |

### Tool Schema Example

From the test file, here's what tool definitions look like:

```java
// Tool: getTopics
new McpSchema.Tool(
    "getTopics",                           // Tool name
    "getTopics",                           // Description
    new McpSchema.JsonSchema(
        "object",
        Map.of(
            "clusterName", Map.of("type", "string"),
            "page", Map.of("type", "integer"),
            "perPage", Map.of("type", "integer"),
            "showInternal", Map.of("type", "boolean"),
            "search", Map.of("type", "string"),
            "orderBy", /* TopicColumnsToSortDTO schema */,
            "sortOrder", /* SortOrderDTO schema */
        ),
        List.of("clusterName"),            // Required parameters
        false, null, null
    )
)
```

**JSON equivalent:**
```json
{
  "name": "getTopics",
  "description": "getTopics",
  "inputSchema": {
    "type": "object",
    "properties": {
      "clusterName": { "type": "string" },
      "page": { "type": "integer" },
      "perPage": { "type": "integer" },
      "showInternal": { "type": "boolean" },
      "search": { "type": "string" }
    },
    "required": ["clusterName"]
  }
}
```

---

## Transport Layer

### SSE (Server-Sent Events)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SSE Transport Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   MCP Client                           Kafka UI MCP Server      │
│   ──────────                           ────────────────────     │
│       │                                        │                │
│       │──── POST /mcp/message ────────────────▶│                │
│       │     {tool: "getTopics", args: {...}}   │                │
│       │                                        │                │
│       │◀──── SSE /mcp/sse ─────────────────────│                │
│       │      {result: [...topics...]}          │                │
│       │                                        │                │
│       │──── POST /mcp/message ────────────────▶│                │
│       │     {tool: "createTopic", args: {...}} │                │
│       │                                        │                │
│       │◀──── SSE /mcp/sse ─────────────────────│                │
│       │      {result: {success: true}}         │                │
│       │                                        │                │
└─────────────────────────────────────────────────────────────────┘
```

### Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/mcp/message` | POST | Send tool invocation requests |
| `/mcp/sse` | GET | Receive server events (results, notifications) |

---

## Dependencies

### Gradle Configuration

```toml
# gradle/libs.versions.toml
modelcontextprotocol-spring-webflux = {
  module = 'io.modelcontextprotocol.sdk:mcp-spring-webflux', 
  version = '0.10.0'
}
```

### Maven Equivalent

```xml
<dependency>
    <groupId>io.modelcontextprotocol.sdk</groupId>
    <artifactId>mcp-spring-webflux</artifactId>
    <version>0.10.0</version>
</dependency>
```

---

## Usage Examples

### Connecting with an MCP Client

```javascript
// Example: Using MCP client to interact with Kafka UI
const mcpClient = new McpClient({
  transport: new SseTransport({
    messageUrl: 'http://localhost:8080/mcp/message',
    sseUrl: 'http://localhost:8080/mcp/sse'
  })
});

// List available tools
const tools = await mcpClient.listTools();

// Call a tool
const topics = await mcpClient.callTool('getTopics', {
  clusterName: 'production-cluster',
  showInternal: false
});
```

### AI Agent Conversation Example

```
User: "What topics exist in my production cluster?"

AI Agent: Let me check the Kafka cluster for you.
          [Calls MCP tool: getTopics({clusterName: "production"})]
          
          Found 15 topics:
          - orders (3 partitions, RF=3)
          - payments (6 partitions, RF=3)
          - notifications (3 partitions, RF=2)
          ...

User: "Create a new topic called 'events' with 6 partitions"

AI Agent: Creating the topic now.
          [Calls MCP tool: createTopic({
            clusterName: "production",
            topicName: "events",
            partitions: 6,
            replicationFactor: 3
          })]
          
          ✓ Topic 'events' created successfully.
```

---

## Configuring Cursor IDE to Use Kafka UI MCP

### Prerequisites

1. Kafka UI must be running with MCP enabled (`mcp.enabled=true`)
2. The server must be accessible (default: `http://localhost:8080`)

### Step 1: Open Cursor Settings

```
Cursor → Settings → Cursor Settings → MCP
```

Or use keyboard shortcut: `Cmd+Shift+P` → "Cursor Settings" → Navigate to MCP section

### Step 2: Add MCP Server Configuration

Click "Add new MCP server" and configure:

```json
{
  "mcpServers": {
    "kafka-ui": {
      "url": "http://localhost:8080/mcp/sse",
      "transport": "sse"
    }
  }
}
```

### Alternative: Edit `~/.cursor/mcp.json` Directly

```json
{
  "mcpServers": {
    "kafka-ui": {
      "url": "http://localhost:8080/mcp/sse",
      "transport": "sse"
    }
  }
}
```

### Step 3: Verify Connection

Once configured, Cursor will:
1. Connect to the SSE endpoint at `/mcp/sse`
2. Discover available tools automatically
3. Show Kafka UI tools in the MCP tools panel

### Configuration for Different Environments

```json
{
  "mcpServers": {
    "kafka-ui-local": {
      "url": "http://localhost:8080/mcp/sse",
      "transport": "sse"
    },
    "kafka-ui-dev": {
      "url": "http://dev-kafka-ui.internal:8080/mcp/sse",
      "transport": "sse"
    },
    "kafka-ui-prod": {
      "url": "https://kafka-ui.company.com/mcp/sse",
      "transport": "sse"
    }
  }
}
```

### Visual: Cursor MCP Setup Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Cursor MCP Configuration                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Set Java version and start Kafka UI with MCP enabled                │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ $ sdk use java 25.0.2-zulu                                   │    │
│     │ $ ./gradlew :api:bootRun --args='--spring.profiles.active=dev'│ │
│     │ ...                                                          │    │
│     │ MCP Server started at /mcp/sse                               │    │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                          │
│                              ▼                                          │
│  2. Configure Cursor                                                    │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ Cursor Settings → MCP → Add Server                          │     │
│     │                                                              │    │
│     │   Name: kafka-ui                                             │    │
│     │   URL:  http://localhost:8080/mcp/sse                        │    │
│     │   Transport: sse                                             │    │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                          │
│                              ▼                                          │
│  3. Use MCP Tools                                                       │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ User: "List all topics in local cluster"                    │     │
│     │                                                              │    │
│     │ Cursor: [Calls kafka-ui.getTopics]                          │     │
│     │         Found 5 topics: orders, payments, users...          │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| "Connection refused" | Ensure Kafka UI is running and MCP is enabled |
| "No tools discovered" | Check that `mcp.enabled=true` in application config |
| "SSE timeout" | Verify firewall/proxy allows SSE connections |
| Tools not appearing | Restart Cursor after adding MCP config |

### Using MCP Tools in Cursor

Once connected, you can ask Cursor to perform Kafka operations:

```
"List all topics in the local cluster"
"Create a topic called 'events' with 6 partitions"
"Show consumer groups for the production cluster"
"What's the lag for consumer group 'my-app'?"
"Delete topic 'test-topic' from local cluster"
```

Cursor will automatically invoke the appropriate MCP tools.

---

## Server Capabilities

The MCP server advertises these capabilities:

```java
var capabilities = McpSchema.ServerCapabilities.builder()
    .resources(false, true)  // Resources: read-only, subscribable
    .tools(true)             // Tools: enabled
    .prompts(false)          // Prompts: not supported
    .logging()               // Logging: enabled
    .build();
```

| Capability | Status | Description |
|------------|--------|-------------|
| Tools | ✅ Enabled | Full tool support for Kafka operations |
| Resources | 📖 Read-only | Resource subscriptions supported |
| Prompts | ❌ Disabled | No prompt templates |
| Logging | ✅ Enabled | Server logging available |

---

## File References

| File | Purpose |
|------|---------|
| `api/src/main/java/io/kafbat/ui/config/McpConfig.java` | MCP server configuration and setup |
| `api/src/main/java/io/kafbat/ui/service/mcp/McpTool.java` | Marker interface for MCP-enabled controllers |
| `api/src/main/java/io/kafbat/ui/service/mcp/McpSpecificationGenerator.java` | Converts controllers to MCP tool specifications |
| `api/src/test/java/io/kafbat/ui/service/mcp/McpSpecificationGeneratorTest.java` | Unit tests for specification generation |
| `gradle/libs.versions.toml` | MCP SDK dependency definition |

---

## External Documentation

- [Kafka UI MCP Documentation](https://ui.docs.kafbat.io/faq/mcp)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [MCP Java SDK](https://github.com/modelcontextprotocol/java-sdk)

---

*Last updated: February 2026*
