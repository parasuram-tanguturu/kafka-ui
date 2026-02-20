# Project Brief: Kafbat UI

## Overview

**Kafbat UI** is a free, open-source web UI for monitoring and managing Apache Kafka® clusters. It provides an intuitive interface for developers and operations teams to manage topics, view messages, monitor brokers, and configure Kafka ecosystem components.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           KAFBAT UI                                     │
│                                                                         │
│   "Versatile, fast and lightweight web UI for managing Apache Kafka"   │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Topics    │  │  Brokers    │  │  Consumers  │  │   Schema    │    │
│  │  Management │  │  Overview   │  │   Groups    │  │  Registry   │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Message   │  │   Kafka     │  │    KSQL     │  │    ACL      │    │
│  │   Browser   │  │   Connect   │  │    DB       │  │  Management │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Requirements

### Functional Requirements

| Feature | Description | Priority |
|---------|-------------|----------|
| Multi-Cluster Management | Manage multiple Kafka clusters from single UI | HIGH |
| Topic Management | Create, configure, delete topics | HIGH |
| Message Browser | View messages in JSON, Avro, Protobuf formats | HIGH |
| Consumer Groups | Monitor consumer lag, offsets | HIGH |
| Schema Registry | Manage Avro, JSON Schema, Protobuf schemas | HIGH |
| Kafka Connect | Manage connectors lifecycle | MEDIUM |
| KSQL Integration | Execute KSQL queries | MEDIUM |
| ACL Management | Configure access control lists | MEDIUM |
| RBAC | Role-based access control for UI users | HIGH |
| Metrics Dashboard | Real-time Kafka metrics visualization | HIGH |

### Non-Functional Requirements

| Requirement | Target | Notes |
|-------------|--------|-------|
| Response Time | < 500ms | For typical UI operations |
| Concurrent Users | 100+ | Per instance |
| Browser Support | Modern browsers | Chrome, Firefox, Safari, Edge |
| Deployment | Docker, Kubernetes | Helm charts available |
| Security | OAuth 2.0, LDAP, Basic Auth | Multiple auth providers |

## Project Boundaries

### In Scope
- Web-based UI for Kafka administration
- Read/write operations on Kafka resources
- Integration with Schema Registry
- Integration with Kafka Connect
- KSQL query execution
- Multi-cluster support
- Authentication & Authorization

### Out of Scope
- Kafka broker deployment/management
- Schema Registry server hosting
- Kafka Connect server hosting
- Data pipeline orchestration
- Long-term metrics storage (delegates to Prometheus)

## Success Criteria

```
┌────────────────────────────────────────────────────────────────────┐
│                      SUCCESS METRICS                                │
├─────────────────────────┬──────────────────────────────────────────┤
│ User Adoption           │ Growing GitHub stars (40k+)              │
│ Community Engagement    │ Active Discord community                 │
│ Stability               │ < 1% error rate in production            │
│ Performance             │ UI loads in < 2 seconds                  │
│ Compatibility           │ Works with Kafka 2.x, 3.x                │
└─────────────────────────┴──────────────────────────────────────────┘
```

## Key Stakeholders

- **Kafka Administrators**: Primary users managing clusters
- **Developers**: Debugging message flows, testing producers/consumers
- **DevOps Teams**: Monitoring cluster health
- **Security Teams**: Managing ACLs and access

## Project History

- Originally developed by Provectus
- Forked and maintained by Kafbat team (key original contributors)
- Renamed from "kafka-ui" to "kafbat-ui"
- Continuous development with regular releases

## Documentation Links

- [Official Docs](https://ui.docs.kafbat.io/)
- [Quick Start](https://ui.docs.kafbat.io/quick-start/demo-run)
- [Configuration](https://ui.docs.kafbat.io/configuration/configuration-file)
- [GitHub Repository](https://github.com/kafbat/kafka-ui)
