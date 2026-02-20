# System Patterns: Kafbat UI

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KAFBAT UI ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                              BROWSER (Client)                                │   │
│   │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│   │  │                         React SPA (TypeScript)                       │   │   │
│   │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │   │
│   │  │  │ Cluster │  │  Topic  │  │ Consumer│  │ Schema  │  │ Connect │   │   │   │
│   │  │  │  Views  │  │  Views  │  │  Views  │  │  Views  │  │  Views  │   │   │   │
│   │  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘   │   │   │
│   │  │       └──────────┬─┴──────────┬─┴──────────┬─┴──────────┬─┘        │   │   │
│   │  │                  ▼            ▼            ▼            ▼          │   │   │
│   │  │            ┌───────────────────────────────────────────────┐       │   │   │
│   │  │            │           TanStack Query (State)              │       │   │   │
│   │  │            │         + Generated API Client                │       │   │   │
│   │  │            └─────────────────────┬─────────────────────────┘       │   │   │
│   │  └──────────────────────────────────┼─────────────────────────────────┘   │   │
│   └─────────────────────────────────────┼─────────────────────────────────────┘   │
│                                         │ HTTP/SSE                                │
│   ┌─────────────────────────────────────▼─────────────────────────────────────┐   │
│   │                        SPRING BOOT BACKEND (JVM)                           │   │
│   │  ┌─────────────────────────────────────────────────────────────────────┐  │   │
│   │  │                        REST Controllers                              │  │   │
│   │  │   Clusters | Topics | Messages | Consumers | Schemas | Connect      │  │   │
│   │  └─────────────────────────────────────────────────────────────────────┘  │   │
│   │                                    │                                       │   │
│   │  ┌─────────────────────────────────▼───────────────────────────────────┐  │   │
│   │  │                         Service Layer                                │  │   │
│   │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │  │   │
│   │  │  │ Cluster  │ │  Topic   │ │ Consumer │ │  Schema  │ │  Kafka   │  │  │   │
│   │  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Connect  │  │  │   │
│   │  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │  │   │
│   │  └───────┼────────────┼────────────┼────────────┼────────────┼────────┘  │   │
│   │          │            │            │            │            │           │   │
│   │  ┌───────▼────────────▼────────────▼────────────▼────────────▼────────┐  │   │
│   │  │                      Kafka Admin Client Pool                        │  │   │
│   │  └─────────────────────────────────────────────────────────────────────┘  │   │
│   └───────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                         │
└─────────────────────────────────────────┼─────────────────────────────────────────┘
                                          │
              ┌───────────────────────────┼───────────────────────────┐
              │                           │                           │
              ▼                           ▼                           ▼
    ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
    │  Kafka Cluster  │       │ Schema Registry │       │  Kafka Connect  │
    │                 │       │                 │       │                 │
    │  ┌───────────┐  │       │  ┌───────────┐  │       │  ┌───────────┐  │
    │  │  Broker   │  │       │  │  Schemas  │  │       │  │ Connectors│  │
    │  │  Broker   │  │       │  │  Subjects │  │       │  │   Tasks   │  │
    │  │  Broker   │  │       │  └───────────┘  │       │  └───────────┘  │
    │  └───────────┘  │       │                 │       │                 │
    └─────────────────┘       └─────────────────┘       └─────────────────┘
```

## Module Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              PROJECT MODULES                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   kafbat-ui/ (Root Project - Gradle Multi-Module)                                   │
│   │                                                                                  │
│   ├── api/                          ← Spring Boot Backend Application               │
│   │   ├── src/main/java/            ← Java source code                              │
│   │   │   └── io.kafbat.ui/                                                         │
│   │   │       ├── controller/       ← REST API Controllers                          │
│   │   │       ├── service/          ← Business logic                                │
│   │   │       ├── model/            ← Domain models                                 │
│   │   │       ├── config/           ← Spring configuration                          │
│   │   │       └── exception/        ← Custom exceptions                             │
│   │   └── Dockerfile                ← Container definition                          │
│   │                                                                                  │
│   ├── frontend/                     ← React TypeScript SPA                          │
│   │   ├── src/                                                                       │
│   │   │   ├── components/           ← Reusable UI components                        │
│   │   │   ├── widgets/              ← Feature-specific widgets                      │
│   │   │   ├── lib/                  ← Utilities, hooks, types                       │
│   │   │   ├── generated-sources/    ← OpenAPI generated API client                  │
│   │   │   └── theme/                ← Styling and theming                           │
│   │   └── package.json              ← npm dependencies (pnpm)                       │
│   │                                                                                  │
│   ├── contract/                     ← API Contract (OpenAPI spec)                   │
│   │   └── build.gradle              ← Generates server stubs                        │
│   │                                                                                  │
│   ├── contract-typespec/            ← TypeSpec API Definitions                      │
│   │   └── api/                      ← TypeSpec source files                         │
│   │                                                                                  │
│   ├── serde-api/                    ← Serialization/Deserialization API             │
│   │   └── src/main/java/            ← Plugin API for custom serdes                  │
│   │                                                                                  │
│   ├── e2e-playwright/               ← End-to-End Tests                              │
│   │   └── Dockerfile                ← Playwright test container                     │
│   │                                                                                  │
│   └── documentation/                ← Documentation and examples                    │
│       └── compose/                  ← Docker Compose examples                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Key Design Patterns

### 1. Contract-First API Design

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           CONTRACT-FIRST WORKFLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│     ┌─────────────┐          ┌─────────────┐          ┌─────────────┐              │
│     │  TypeSpec   │  build   │   OpenAPI   │ generate │   Server    │              │
│     │ Definition  │────────▶ │    Spec     │────────▶ │   Stubs     │              │
│     │  (.tsp)     │          │   (.yaml)   │          │  (Java)     │              │
│     └─────────────┘          └──────┬──────┘          └─────────────┘              │
│                                     │                                               │
│                                     │ generate                                      │
│                                     ▼                                               │
│                              ┌─────────────┐                                        │
│                              │   Client    │                                        │
│                              │    SDK      │                                        │
│                              │ (TypeScript)│                                        │
│                              └─────────────┘                                        │
│                                                                                      │
│  Benefits:                                                                           │
│  • Single source of truth for API                                                   │
│  • Auto-generated client code                                                        │
│  • Type safety across frontend/backend                                              │
│  • API documentation always in sync                                                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2. Reactive Programming (Backend)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      REACTIVE STREAM PATTERN (WebFlux)                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│    Traditional (Blocking)              Reactive (Non-Blocking)                       │
│    ─────────────────────              ───────────────────────                       │
│                                                                                      │
│    Request ─────────┐                 Request ─────────┐                            │
│                     ▼                                  ▼                            │
│    Thread 1: [========= Kafka Call =========]         Thread 1: [subscribe] →       │
│                                                                     │               │
│    Request ─────────┐                                              ▼               │
│                     ▼                                    Kafka: [========]           │
│    Thread 2: [========= Kafka Call =========]                      │               │
│                                                                    ▼               │
│    Request ─────────┐                                    Thread: [emit] →           │
│                     ▼                                                               │
│    Thread 3: [========= Kafka Call =========]                                       │
│                                                                                      │
│    Threads needed: 3                   Threads needed: 1                            │
│    Memory usage: HIGH                  Memory usage: LOW                            │
│                                                                                      │
│  Code Pattern:                                                                       │
│  ─────────────                                                                      │
│  public Mono<ResponseEntity<TopicDTO>> getTopic(String name) {                      │
│      return topicService.getTopic(name)     // Returns Mono<Topic>                  │
│          .map(this::mapToDTO)                // Transform                           │
│          .map(ResponseEntity::ok);           // Wrap response                       │
│  }                                                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3. Server-Sent Events (Message Streaming)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         MESSAGE POLLING ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   Browser                      Backend                         Kafka                 │
│   ───────                      ───────                         ─────                 │
│      │                            │                              │                   │
│      │  SSE Connection            │                              │                   │
│      │──────────────────────────▶ │                              │                   │
│      │                            │                              │                   │
│      │                            │  Start Consumer              │                   │
│      │                            │────────────────────────────▶ │                   │
│      │                            │                              │                   │
│      │  event: message            │  ◀─────────── record ─────── │                   │
│      │◀────────────────────────── │                              │                   │
│      │                            │                              │                   │
│      │  event: message            │  ◀─────────── record ─────── │                   │
│      │◀────────────────────────── │                              │                   │
│      │                            │                              │                   │
│      │  event: phase:consuming    │                              │                   │
│      │◀────────────────────────── │                              │                   │
│      │                            │                              │                   │
│      │  Close                     │  Close Consumer              │                   │
│      │──────────────────────────▶ │────────────────────────────▶ │                   │
│      │                            │                              │                   │
│                                                                                      │
│  Message Event Format:                                                               │
│  ─────────────────────                                                              │
│  {                                                                                   │
│    "type": "MESSAGE",                                                               │
│    "message": {                                                                      │
│      "partition": 0,                                                                │
│      "offset": 12345,                                                               │
│      "timestamp": "2024-01-15T10:30:00Z",                                           │
│      "key": "order-123",                                                            │
│      "content": { "orderId": "123", "amount": 99.99 }                               │
│    }                                                                                │
│  }                                                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4. Role-Based Access Control (RBAC)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              RBAC ARCHITECTURE                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                           AUTHENTICATION LAYER                               │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │   │
│   │  │ OAuth2  │  │  LDAP   │  │  Basic  │  │ AWS IAM │  │ GCP IAM │           │   │
│   │  │ GitHub  │  │         │  │  Auth   │  │         │  │         │           │   │
│   │  │ Google  │  │         │  │         │  │         │  │         │           │   │
│   │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘           │   │
│   │       └──────────┬─┴──────────┬─┴──────────┬─┴──────────┬─┘                │   │
│   │                  ▼            ▼            ▼            ▼                  │   │
│   │            ┌───────────────────────────────────────────────┐               │   │
│   │            │              Authenticated User               │               │   │
│   │            │         (username, groups, claims)            │               │   │
│   │            └─────────────────────┬─────────────────────────┘               │   │
│   └──────────────────────────────────┼─────────────────────────────────────────┘   │
│                                      │                                              │
│   ┌──────────────────────────────────▼─────────────────────────────────────────┐   │
│   │                           AUTHORIZATION LAYER                               │   │
│   │                                                                             │   │
│   │   Role Definition (YAML):           Access Check Flow:                      │   │
│   │   ─────────────────────            ───────────────────                      │   │
│   │   roles:                            Request                                 │   │
│   │     - name: "admin"                   │                                     │   │
│   │       subjects:                       ▼                                     │   │
│   │         - type: "user"              ┌────────────┐                          │   │
│   │           value: "alice"            │ Get User   │                          │   │
│   │       permissions:                  │   Roles    │                          │   │
│   │         - resource: topic           └─────┬──────┘                          │   │
│   │           value: "*"                      │                                 │   │
│   │           actions:                        ▼                                 │   │
│   │             - all                   ┌────────────┐                          │   │
│   │                                     │ Check      │──▶ Allowed/Denied        │   │
│   │     - name: "developer"             │ Permission │                          │   │
│   │       subjects:                     └────────────┘                          │   │
│   │         - type: "group"                                                     │   │
│   │           value: "developers"                                               │   │
│   │       permissions:                                                          │   │
│   │         - resource: topic                                                   │   │
│   │           value: "dev-*"                                                    │   │
│   │           actions: [VIEW, MESSAGES_READ]                                    │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│   Permission Types:                                                                  │
│   ─────────────────                                                                 │
│   ┌─────────────┬────────────────────────────────────────────────────────────────┐  │
│   │ Resource    │ Actions                                                         │  │
│   ├─────────────┼────────────────────────────────────────────────────────────────┤  │
│   │ CLUSTER     │ VIEW, EDIT                                                      │  │
│   │ TOPIC       │ VIEW, CREATE, EDIT, DELETE, MESSAGES_READ, MESSAGES_PRODUCE    │  │
│   │ CONSUMER    │ VIEW, DELETE, RESET_OFFSETS                                     │  │
│   │ SCHEMA      │ VIEW, CREATE, DELETE, MODIFY_GLOBAL_COMPAT                      │  │
│   │ CONNECT     │ VIEW, EDIT, CREATE_CONNECTOR, DELETE_CONNECTOR                  │  │
│   │ KSQL        │ EXECUTE                                                         │  │
│   │ ACL         │ VIEW, EDIT                                                      │  │
│   └─────────────┴────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 5. Data Masking

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA MASKING FLOW                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   Original Message                    Masked Message (to UI)                         │
│   ────────────────                    ─────────────────────                          │
│   {                                   {                                              │
│     "user": "john@email.com",           "user": "j***@email.com",                    │
│     "ssn": "123-45-6789",               "ssn": "***-**-****",                        │
│     "amount": 1500.00,                  "amount": 1500.00,                           │
│     "creditCard": "4111111111111111"    "creditCard": "411111******1111"             │
│   }                                   }                                              │
│                                                                                      │
│   Configuration:                                                                     │
│   ──────────────                                                                    │
│   masking:                                                                           │
│     rules:                                                                           │
│       - type: MASK                                                                   │
│         topicKeysPattern: ".*sensitive.*"                                           │
│         fields: ["ssn", "creditCard"]                                               │
│         pattern: "mask-except-last-4"                                               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Component Relationships

### Backend Services

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           SERVICE DEPENDENCY GRAPH                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│                              ┌────────────────────┐                                 │
│                              │  AbstractController │                                 │
│                              │   (Base class)      │                                 │
│                              └──────────┬─────────┘                                 │
│                    ┌────────────────────┼────────────────────┐                      │
│                    │                    │                    │                      │
│              ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐                 │
│              │  Cluster  │       │   Topic   │       │  Message  │                 │
│              │ Controller│       │ Controller│       │ Controller│                 │
│              └─────┬─────┘       └─────┬─────┘       └─────┬─────┘                 │
│                    │                   │                   │                        │
│              ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐                 │
│              │  Cluster  │       │   Topic   │       │  Message  │                 │
│              │  Service  │       │  Service  │       │  Service  │                 │
│              └─────┬─────┘       └─────┬─────┘       └─────┬─────┘                 │
│                    │                   │                   │                        │
│                    └─────────────┬─────┴──────────┬────────┘                        │
│                                  │                │                                 │
│                           ┌──────▼──────┐   ┌─────▼──────┐                          │
│                           │   Cluster   │   │ Deserialization │                     │
│                           │    Cache    │   │   Service  │                          │
│                           └──────┬──────┘   └────────────┘                          │
│                                  │                                                  │
│                           ┌──────▼──────┐                                           │
│                           │ KafkaAdmin  │                                           │
│                           │   Client    │                                           │
│                           └─────────────┘                                           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Frontend Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          FRONTEND COMPONENT STRUCTURE                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   App                                                                                │
│   │                                                                                  │
│   ├── Layout                                                                         │
│   │   ├── Sidebar (Cluster Navigation)                                              │
│   │   └── Main Content Area                                                         │
│   │                                                                                  │
│   └── Routes                                                                         │
│       │                                                                              │
│       ├── /clusters                                                                  │
│       │   └── ClustersList                                                           │
│       │                                                                              │
│       ├── /clusters/:name                                                            │
│       │   └── ClusterOverview                                                        │
│       │       ├── BrokersWidget                                                      │
│       │       ├── TopicsWidget                                                       │
│       │       └── MetricsWidget                                                      │
│       │                                                                              │
│       ├── /clusters/:name/topics                                                     │
│       │   └── TopicsList                                                             │
│       │       └── TopicItem (repeated)                                               │
│       │                                                                              │
│       ├── /clusters/:name/topics/:topic                                              │
│       │   └── TopicDetails                                                           │
│       │       ├── TopicOverview                                                      │
│       │       ├── TopicMessages     ← Uses SSE for live messages                    │
│       │       ├── TopicConsumers                                                     │
│       │       └── TopicSettings                                                      │
│       │                                                                              │
│       ├── /clusters/:name/consumer-groups                                            │
│       │   └── ConsumerGroupsList                                                     │
│       │                                                                              │
│       ├── /clusters/:name/schemas                                                    │
│       │   └── SchemasList                                                            │
│       │       └── SchemaDetails                                                      │
│       │                                                                              │
│       └── /clusters/:name/kafka-connect                                              │
│           └── ConnectorsList                                                         │
│               └── ConnectorDetails                                                   │
│                                                                                      │
│   Shared Components:                                                                 │
│   ─────────────────                                                                 │
│   components/                                                                        │
│   ├── common/        ← Buttons, Inputs, Tables, Modals                              │
│   ├── ACLPage/       ← ACL management components                                    │
│   ├── Brokers/       ← Broker-specific components                                   │
│   ├── Topics/        ← Topic-related components                                     │
│   └── ConsumerGroups/← Consumer group components                                    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Design Decisions Log

| Decision | Rationale | Trade-offs |
|----------|-----------|------------|
| WebFlux over MVC | High concurrency with Kafka polling, SSE support | Steeper learning curve, debugging complexity |
| TypeSpec for API | Type-safe contracts, better tooling | Additional build step |
| React + TypeScript | Type safety, modern ecosystem | Larger bundle size |
| styled-components | Component-scoped CSS, dynamic theming | Runtime overhead |
| TanStack Query | Powerful caching, automatic refetching | Additional dependency |
| Gradle multi-module | Clear separation, shared dependencies | Complex build configuration |
| Docker-first deployment | Consistent environments, easy scaling | Container overhead |
