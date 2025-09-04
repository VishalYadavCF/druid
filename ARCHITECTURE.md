# Apache Druid Architecture Summary

## Overview

Apache Druid is a high-performance, real-time analytics database designed for fast slice-and-dice analytics (OLAP queries) on large datasets. Druid excels at powering interactive applications, operational dashboards, and high-concurrency APIs that require sub-second to few-second query response times.

**Key Value Proposition**: Reduce time to insight and action by combining the best of data warehouses, timeseries databases, and search systems into a unified, cloud-native platform.

## Core Design Principles

1. **Real-time & Batch**: Supports both streaming and batch data ingestion with immediate query availability
2. **Columnar Storage**: Optimized column-oriented storage for fast analytical queries
3. **Distributed & Scalable**: Horizontal scaling across hundreds of servers with automatic load balancing
4. **Self-Healing**: Fault-tolerant architecture with automatic recovery and rebalancing
5. **Time-Optimized**: First-class time-based partitioning and filtering for time-series workloads
6. **Cloud-Native**: Designed for cloud deployment with deep storage integration

## High-Level Architecture

Druid follows a distributed microservices architecture organized into three main server types:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Master Server │    │   Query Server  │    │   Data Server   │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│  • Coordinator  │    │    • Broker     │    │   • Historical  │
│  • Overlord     │    │    • Router     │    │ • Middle Manager│
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │                            │                            │
┌───▼────┐              ┌────────▼────────┐         ┌────────▼────┐
│Deep    │              │   Metadata      │         │  ZooKeeper  │
│Storage │              │   Storage       │         │             │
│(S3/HDFS)│              │ (MySQL/PostgreSQL)│         │             │
└────────┘              └─────────────────┘         └─────────────┘
```

## Core Services

### Master Server Services

#### Coordinator
- **Purpose**: Manages data availability and load balancing across the cluster
- **Responsibilities**:
  - Monitors Historical servers and assigns segments
  - Balances segment distribution for optimal query performance
  - Manages segment lifecycle (loading, dropping, replication)
  - Handles compaction policies and data retention rules

#### Overlord  
- **Purpose**: Controls data ingestion workloads and task management
- **Responsibilities**:
  - Manages ingestion tasks and their lifecycle
  - Coordinates with Middle Managers for task execution
  - Monitors task progress and handles failures
  - Manages ingestion job scheduling and resource allocation

### Query Server Services

#### Broker
- **Purpose**: Query router and result aggregator for external clients
- **Responsibilities**:
  - Receives queries from clients and routes to appropriate servers
  - Merges partial results from multiple data servers
  - Maintains query routing metadata and caches
  - Provides the primary query endpoint for applications

#### Router
- **Purpose**: Unified API gateway and web console host
- **Responsibilities**:  
  - Routes requests to appropriate Brokers, Coordinators, and Overlords
  - Hosts the web console UI for cluster management
  - Provides authentication and authorization (when configured)
  - Load balances across multiple brokers

### Data Server Services

#### Historical
- **Purpose**: Stores and serves queryable historical data segments  
- **Responsibilities**:
  - Downloads immutable segments from deep storage
  - Serves queries against local segment cache
  - Maintains segment metadata and indexes
  - Provides high-throughput query processing

#### Middle Manager
- **Purpose**: Executes data ingestion tasks
- **Responsibilities**:
  - Runs Peon processes for individual ingestion tasks
  - Manages task resources and isolation
  - Coordinates with Overlord for task lifecycle
  - Publishes completed segments to deep storage

##### Peon (Sub-process)
- Individual task execution processes spawned by Middle Manager
- Isolates ingestion tasks for resource management and fault tolerance
- Handles the actual data processing and segment creation

### Alternative: Indexer Service
- **Purpose**: Simplified alternative to Middle Manager + Peon architecture
- **Use Case**: Smaller deployments or container-based environments
- **Benefits**: Reduced resource overhead and simpler deployment model

## Data Flow Architecture

### Ingestion Flow
```
Data Sources → Middle Manager/Indexer → Segment Creation → Deep Storage → Historical Loading
     │              │                        │               │              │
     │              ▼                        │               │              ▼
Kafka/Files    Task Execution           Metadata Store    S3/HDFS      Query Availability
```

### Query Flow  
```
Client → Router → Broker → Historical Servers → Segment Processing → Result Aggregation → Response
```

## Storage Architecture

### Segments
- **Immutable Data Units**: Time-partitioned, compressed columnar data files
- **Structure**: Contains indexes, column data, and metadata for a time interval
- **Optimization**: Automatic summarization and compression at ingestion time

### Deep Storage
- **Purpose**: Permanent, fault-tolerant segment storage
- **Options**: AWS S3, Google Cloud Storage, Azure Blob, HDFS, local filesystem
- **Durability**: Primary source of truth for all data segments

### Metadata Storage
- **Purpose**: Stores cluster metadata, configurations, and segment information
- **Options**: MySQL, PostgreSQL, Apache Derby (dev only)
- **Contents**: Datasource schemas, segment metadata, rules, configurations

### ZooKeeper
- **Purpose**: Service discovery and coordination
- **Usage**: Leader election, service announcements, configuration distribution
- **Critical**: Required for cluster coordination and service discovery

## Technology Stack

### Backend (Java)
- **Language**: Java 11/17
- **Build System**: Apache Maven  
- **Frameworks**: 
  - Guice for dependency injection
  - Jersey for REST APIs
  - Jetty for embedded web server
  - Jackson for JSON processing
  - Caffeine for caching

### Web Console (TypeScript/React)
- **Language**: TypeScript/JavaScript
- **Framework**: React 18
- **Build Tools**: Webpack, Babel
- **UI Components**: Blueprint.js
- **Testing**: Jest, Playwright for E2E

### External Dependencies
- **ZooKeeper**: Service coordination (3.8+)
- **Deep Storage**: S3, HDFS, GCS, Azure Blob, or NFS
- **Metadata DB**: MySQL 8+ or PostgreSQL 10+
- **Message Queues**: Apache Kafka (for streaming ingestion)

## Module Organization

### Core Processing Modules
- **`processing/`**: Core data processing, query execution, and segment handling
- **`server/`**: Common server infrastructure, HTTP endpoints, and service framework  
- **`services/`**: Individual service implementations (Broker, Coordinator, etc.)
- **`sql/`**: SQL parsing, planning, and execution layer
- **`indexing-service/`**: Data ingestion and task management

### Extension Modules  
- **`extensions-core/`**: Essential extensions (AWS, Azure, Kafka, Parquet, etc.)
- **`extensions-contrib/`**: Community-contributed extensions (experimental features)

### Supporting Modules
- **`web-console/`**: TypeScript/React web management interface
- **`integration-tests/`**: End-to-end integration test suites
- **`distribution/`**: Packaging and deployment artifacts
- **`docs/`**: Comprehensive documentation and tutorials

## Extension System

### Core Extensions (Production Ready)
- **Storage**: S3, Azure, Google Cloud, HDFS
- **Ingestion**: Kafka, Kinesis, Avro, Parquet, ORC  
- **Security**: Kerberos, Basic Auth, PAC4J
- **Caching**: Caffeine, Redis
- **Monitoring**: DataSketches, Stats, Histogram

### Contrib Extensions (Community)  
- **Experimental Features**: Delta Lake, Iceberg, gRPC queries
- **Additional Integrations**: InfluxDB, OpenTelemetry, Prometheus
- **Specialized Storage**: Cassandra, Redis cache
- **Custom Analytics**: Moving averages, distinct count algorithms

## Common Deployment Patterns

### Small/Development (Single Node)
- All services on one machine
- Embedded ZooKeeper and Derby metadata store
- Local filesystem for deep storage

### Medium Cluster (3-10 nodes)
- Dedicated Master, Query, and Data server roles
- External ZooKeeper cluster (3 nodes)
- Cloud storage (S3/GCS) with managed database

### Large Cluster (10+ nodes)  
- Multiple instances per service type
- Separate clusters for real-time and batch workloads
- Advanced routing and load balancing
- Multi-region deployments for DR

### Cloud-Native (Kubernetes)
- Containerized service deployment
- Helm charts for orchestration
- Auto-scaling based on query load
- Cloud-managed external dependencies

## Key Architectural Benefits

1. **Independent Scaling**: Each service type scales independently based on workload
2. **Fault Isolation**: Service failures don't cascade across the system  
3. **Operational Simplicity**: Self-healing and self-balancing reduce operational overhead
4. **Performance**: Columnar storage and bitmap indexes enable sub-second queries
5. **Flexibility**: Extensive extension system for customization
6. **Cloud Integration**: First-class support for cloud storage and managed services

## Getting Started

For developers new to Druid:

1. **Start Here**: Read the [Quickstart Tutorial](docs/tutorials/index.md)
2. **Architecture Deep Dive**: Explore [Design Documentation](docs/design/)  
3. **Development Setup**: Follow [Build Instructions](docs/development/build.md)
4. **IDE Setup**: Use [IntelliJ Configuration](dev/intellij-setup.md)

## Further Reading

- [Design Documentation](docs/design/): Detailed component specifications
- [Operations Guide](docs/operations/): Production deployment and monitoring
- [API Reference](docs/api-reference/): Complete REST API documentation  
- [Tutorials](docs/tutorials/): Step-by-step learning guides
- [Community](https://druid.apache.org/community/): Join the Druid community

---

*This architecture summary provides a high-level overview of Apache Druid's design and implementation. For detailed technical specifications, refer to the individual documentation files in the `docs/` directory.*