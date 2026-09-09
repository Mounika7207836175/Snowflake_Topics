# Snowflake: Zero-to-Production Learning Roadmap

This roadmap is designed for a learner who already knows most database and SQL prerequisites. We will explain every concept in simple language, use small examples first, and then apply it in a production-style project.

## How We Will Learn

For every topic, we will use this structure:

1. Simple explanation and analogy
2. Why the concept matters
3. Important theory
4. Syntax and examples
5. Hands-on exercise
6. Common mistakes
7. Interview questions
8. How it is used in production

## Phase 0: Quick Prerequisite Check

- [ ] Relational database basics
- [ ] Tables, rows, columns, keys, and constraints
- [ ] OLTP versus OLAP
- [ ] Basic data warehousing concepts
- [ ] Basic cloud concepts
- [ ] Git fundamentals
- [ ] SQL fundamentals
  - [ ] DDL, DML, and DCL
  - [ ] Filtering, grouping, and aggregation
  - [ ] Joins and subqueries
  - [ ] Common table expressions
  - [ ] Window functions
  - [ ] Set operations
  - [ ] Date and string functions
  - [ ] CASE expressions
  - [ ] Views
  - [ ] Transactions
  - [ ] MERGE

## Phase 1: Snowflake Fundamentals

- [ ] What Snowflake is
- [ ] Problems Snowflake solves
- [ ] Cloud data warehouse versus traditional database
- [ ] Snowflake editions
- [ ] Supported cloud providers and regions
- [ ] Organizations and accounts
- [ ] Snowsight web interface
- [ ] Worksheets and workspaces
- [ ] Snowflake object hierarchy
  - [ ] Organization
  - [ ] Account
  - [ ] Database
  - [ ] Schema
  - [ ] Tables and other schema objects
- [ ] Fully qualified object names
- [ ] Sessions and session context
- [ ] Identifiers and case sensitivity
- [ ] Snowflake SQL dialect
- [ ] Metadata and system functions

## Phase 2: Snowflake Architecture

- [ ] Multi-cluster shared-data architecture
- [ ] Storage layer
- [ ] Compute layer
- [ ] Cloud services layer
- [ ] Separation of storage and compute
- [ ] Massively parallel processing
- [ ] Virtual warehouses
- [ ] Columnar storage and compression
- [ ] Micro-partitions
- [ ] Micro-partition metadata
- [ ] Partition pruning
- [ ] Natural data clustering
- [ ] Clustering keys and clustering depth
- [ ] Result cache
- [ ] Warehouse cache
- [ ] Metadata cache
- [ ] How Snowflake processes a query

## Phase 3: Snowflake Database Objects

- [ ] Databases and schemas
- [ ] Standard tables
- [ ] Permanent tables
- [ ] Transient tables
- [ ] Temporary tables
- [ ] Dynamic tables
- [ ] External tables
- [ ] Apache Iceberg tables
- [ ] Hybrid tables
- [ ] Standard views
- [ ] Secure views
- [ ] Materialized views
- [ ] Sequences
- [ ] Stages
- [ ] File formats
- [ ] Pipes
- [ ] Streams
- [ ] Tasks
- [ ] Stored procedures
- [ ] User-defined functions
- [ ] User-defined table functions
- [ ] Tags and policies

## Phase 4: Data Types and Semi-Structured Data

- [ ] Numeric, string, Boolean, and binary types
- [ ] Date and time types
- [ ] Timestamp types and time zones
- [ ] VARIANT
- [ ] OBJECT
- [ ] ARRAY
- [ ] Structured types
- [ ] JSON
- [ ] Avro
- [ ] ORC
- [ ] Parquet
- [ ] XML
- [ ] Loading semi-structured data
- [ ] Dot and bracket notation
- [ ] Type casting
- [ ] FLATTEN
- [ ] Lateral joins
- [ ] Constructing and modifying arrays and objects
- [ ] Schema detection
- [ ] Schema evolution
- [ ] Handling malformed and changing source data

## Phase 5: Data Loading and Unloading

- [ ] Internal and external stages
- [ ] User stages
- [ ] Table stages
- [ ] Named stages
- [ ] Storage integrations
- [ ] File formats and compression
- [ ] PUT, GET, LIST, and REMOVE
- [ ] Bulk loading with COPY INTO
- [ ] Transforming data during loading
- [ ] Validation mode
- [ ] Error handling and rejected records
- [ ] Load history
- [ ] Duplicate-file detection
- [ ] Purging and managing staged files
- [ ] Unloading with COPY INTO location
- [ ] Partitioned unloading
- [ ] File sizing best practices
- [ ] Snowpipe
- [ ] Cloud event notifications
- [ ] Snowpipe REST API
- [ ] Snowpipe Streaming
- [ ] Kafka and other connectors
- [ ] Batch, micro-batch, and streaming ingestion

## Phase 6: Data Transformation and Pipelines

- [ ] ETL versus ELT
- [ ] Raw, staging, curated, and presentation layers
- [ ] Bronze, silver, and gold architecture
- [ ] Idempotent pipeline design
- [ ] Incremental processing
- [ ] Change data capture
- [ ] Standard streams
- [ ] Append-only streams
- [ ] Insert-only streams
- [ ] Stream offsets and staleness
- [ ] Scheduled tasks
- [ ] Triggered tasks
- [ ] Serverless tasks
- [ ] Task graphs and dependencies
- [ ] Task retries and monitoring
- [ ] Dynamic tables
- [ ] Target lag
- [ ] Incremental and full refresh
- [ ] Dynamic-table refresh modes
- [ ] Dynamic-table dependency graphs
- [ ] MERGE and deduplication
- [ ] Snowflake Scripting
- [ ] Pipeline error handling and replay
- [ ] Dynamic tables versus streams and tasks
- [ ] Dynamic tables versus materialized views

## Phase 7: Data Modeling

- [ ] Dimensional modeling
- [ ] Facts and dimensions
- [ ] Star schema
- [ ] Snowflake schema
- [ ] Defining table grain
- [ ] Natural and surrogate keys
- [ ] Slowly changing dimension Type 0
- [ ] Slowly changing dimension Type 1
- [ ] Slowly changing dimension Type 2
- [ ] Slowly changing dimension Type 3
- [ ] Conformed dimensions
- [ ] Transaction fact tables
- [ ] Periodic snapshot fact tables
- [ ] Accumulating snapshot fact tables
- [ ] Data Vault fundamentals
- [ ] Normalization and denormalization trade-offs
- [ ] Modeling semi-structured data
- [ ] Data marts
- [ ] Semantic layers
- [ ] Naming and modeling standards
- [ ] Data quality rules

## Phase 8: Virtual Warehouses and Workload Management

- [ ] Standard warehouses
- [ ] Snowpark-optimized warehouses
- [ ] Warehouse sizes
- [ ] Starting, suspending, and resuming warehouses
- [ ] Resizing warehouses
- [ ] Auto-suspend and auto-resume
- [ ] Scaling up versus scaling out
- [ ] Multi-cluster warehouses
- [ ] Maximized versus auto-scale modes
- [ ] Concurrency and queuing
- [ ] Workload isolation
- [ ] Separate warehouses for ingestion, transformation, BI, and data science
- [ ] Query Acceleration Service
- [ ] Resource monitors
- [ ] Warehouse utilization and right-sizing

## Phase 9: Performance Optimization

- [ ] Query History
- [ ] Query Profile and execution plans
- [ ] Bytes and partitions scanned
- [ ] Partition pruning
- [ ] Exploding joins
- [ ] Data skew
- [ ] Join strategy
- [ ] Filter placement
- [ ] Efficient SQL patterns
- [ ] Avoiding unnecessary scans and SELECT star
- [ ] Warehouse sizing
- [ ] Cache utilization
- [ ] Clustering keys
- [ ] Automatic Clustering
- [ ] Search Optimization Service
- [ ] Materialized views
- [ ] Query Acceleration Service
- [ ] Top-K pruning
- [ ] Diagnosing spills, queuing, and remote I/O
- [ ] Performance tests and baselines
- [ ] Balancing performance and cost

## Phase 10: Security and Access Control

- [ ] Authentication versus authorization
- [ ] Users and service users
- [ ] System-defined roles
- [ ] Custom account roles
- [ ] Database roles
- [ ] Role hierarchy and inheritance
- [ ] Primary and secondary roles
- [ ] Object ownership
- [ ] Privileges and grants
- [ ] Future grants
- [ ] Managed access schemas
- [ ] Least-privilege RBAC design
- [ ] Service accounts
- [ ] Multi-factor authentication
- [ ] Key-pair authentication
- [ ] Single sign-on and federated authentication
- [ ] OAuth
- [ ] SCIM provisioning
- [ ] Network rules and network policies
- [ ] Private connectivity
- [ ] Storage and API integrations
- [ ] Secrets and external access integrations
- [ ] Encryption and key management
- [ ] Tri-Secret Secure
- [ ] Secure views and secure functions
- [ ] Access History and login auditing
- [ ] Periodic access reviews

## Phase 11: Governance, Privacy, and Compliance

- [ ] Snowflake Horizon Catalog
- [ ] Object tagging
- [ ] Tag-based masking
- [ ] Dynamic data masking
- [ ] Row access policies
- [ ] Projection policies
- [ ] Aggregation policies
- [ ] Join policies
- [ ] Password policies
- [ ] Session policies
- [ ] Sensitive-data classification
- [ ] Data lineage
- [ ] Data-quality monitoring
- [ ] Data metric functions
- [ ] Audit and Access History
- [ ] Data-retention policies
- [ ] Separation of duties
- [ ] Regulatory and compliance considerations

## Phase 12: Data Protection and Disaster Recovery

- [ ] Time Travel
- [ ] Data-retention periods
- [ ] UNDROP
- [ ] Fail-safe
- [ ] Zero-copy cloning
- [ ] Cloning databases, schemas, and tables
- [ ] Backup and restore strategy
- [ ] Database replication
- [ ] Replication groups
- [ ] Failover groups
- [ ] Cross-region and cross-cloud considerations
- [ ] Failover and failback
- [ ] Recovery point objective
- [ ] Recovery time objective
- [ ] Business-continuity testing
- [ ] Disaster-recovery runbooks

## Phase 13: Data Sharing and Collaboration

- [ ] Secure Data Sharing
- [ ] Provider and consumer accounts
- [ ] Shares and database roles
- [ ] Reader accounts
- [ ] Listings
- [ ] Snowflake Marketplace
- [ ] Private listings
- [ ] Data Exchange
- [ ] Cross-region sharing
- [ ] Shared-data security
- [ ] Clean Rooms
- [ ] Native Apps fundamentals
- [ ] Provider cost and operational considerations

## Phase 14: Cost Management and FinOps

- [ ] Snowflake credits
- [ ] Compute costs
- [ ] Storage costs
- [ ] Cloud-services costs
- [ ] Serverless-feature costs
- [ ] Data-transfer costs
- [ ] Warehouse billing behavior
- [ ] Auto-suspend configuration
- [ ] Warehouse right-sizing
- [ ] Multi-cluster cost management
- [ ] Resource monitors
- [ ] Budgets
- [ ] Tags and cost attribution
- [ ] ACCOUNT_USAGE cost views
- [ ] Chargeback and showback
- [ ] Storage usage
- [ ] Time Travel and Fail-safe storage costs
- [ ] Automatic Clustering costs
- [ ] Search optimization costs
- [ ] Materialized-view maintenance costs
- [ ] Snowpipe and serverless task costs
- [ ] Cost anomaly detection
- [ ] Cost-performance benchmarking

## Phase 15: Monitoring and Production Operations

- [ ] INFORMATION_SCHEMA
- [ ] Shared SNOWFLAKE database
- [ ] ACCOUNT_USAGE
- [ ] ORGANIZATION_USAGE
- [ ] READER_ACCOUNT_USAGE
- [ ] Query History
- [ ] Login History
- [ ] Load and copy histories
- [ ] Pipe usage and history
- [ ] Task History
- [ ] Dynamic-table refresh history
- [ ] Warehouse load and metering history
- [ ] Alerts and notifications
- [ ] Event tables and logging
- [ ] Snowpark telemetry
- [ ] Operational dashboards
- [ ] Service-level objectives and indicators
- [ ] Pipeline freshness and latency monitoring
- [ ] Incident response
- [ ] Troubleshooting failed loads, tasks, and queries
- [ ] Runbooks and on-call practices
- [ ] Auditing configuration changes

## Phase 16: Connectivity and Application Integration

- [ ] SnowSQL and Snowflake CLI
- [ ] JDBC and ODBC
- [ ] Python connector
- [ ] Node.js and other language connectors
- [ ] SQLAlchemy
- [ ] REST APIs and SQL API
- [ ] Partner Connect
- [ ] Power BI and Tableau
- [ ] ETL and ELT tools
- [ ] dbt with Snowflake
- [ ] Airflow and other orchestrators
- [ ] Kafka connector
- [ ] Connection pooling
- [ ] Authentication and secret handling
- [ ] Retry logic and idempotency
- [ ] Driver configuration and troubleshooting

## Phase 17: Snowpark and Programmability

- [ ] Snowpark DataFrames
- [ ] Snowpark Python
- [ ] Snowpark Java and Scala overview
- [ ] Lazy evaluation
- [ ] Snowpark built-in functions
- [ ] UDFs and UDTFs
- [ ] Stored procedures
- [ ] Packages and dependency management
- [ ] Vectorized Python UDFs
- [ ] File handling
- [ ] Logging and profiling
- [ ] Snowpark-optimized warehouses
- [ ] Snowpark pandas API
- [ ] Snowpark ML fundamentals
- [ ] Model Registry
- [ ] Feature Store
- [ ] Container Services overview

## Phase 18: Snowflake AI Capabilities

- [ ] Cortex AI functions
- [ ] Cortex Analyst
- [ ] Cortex Search
- [ ] Document AI
- [ ] Embeddings and vector data
- [ ] Retrieval-augmented generation
- [ ] AI evaluation and observability
- [ ] AI security and cost controls
- [ ] Building governed AI applications

## Phase 19: DevOps and Infrastructure as Code

- [ ] Development, testing, staging, and production environments
- [ ] Account and environment isolation
- [ ] Git-based workflow
- [ ] Reusable SQL scripts
- [ ] Snowflake CLI projects
- [ ] Infrastructure as code
- [ ] Terraform Snowflake provider
- [ ] dbt project structure
- [ ] CI/CD pipelines
- [ ] Automated validation and testing
- [ ] Database migration tools
- [ ] Parameter and secret management
- [ ] Deployment ordering and dependencies
- [ ] Rollback strategies
- [ ] Zero-copy clones for testing
- [ ] Change management and releases
- [ ] Preventing configuration drift

## Phase 20: Testing and Production Engineering

- [ ] Unit tests
- [ ] Integration tests
- [ ] End-to-end tests
- [ ] Data-quality tests
- [ ] Schema and data-contract tests
- [ ] Reconciliation testing
- [ ] Freshness and completeness tests
- [ ] Pipeline idempotency
- [ ] Late-arriving data
- [ ] Duplicate handling
- [ ] Backfills and replay
- [ ] Failure recovery
- [ ] Performance regression testing
- [ ] Cost regression testing
- [ ] Security testing
- [ ] Deployment validation
- [ ] Technical documentation
- [ ] Production runbooks

## Production Project: E-Commerce Analytics Platform

We will build one project while learning rather than waiting until the end.

### Project Flow

```text
CSV, JSON, API, or streaming events
                 |
                 v
       Stages and COPY/Snowpipe
                 |
                 v
              Raw layer
                 |
                 v
 Dynamic Tables or Streams and Tasks
                 |
                 v
       Clean dimensional model
                 |
                 v
 Sales, customer, inventory, and finance marts
                 |
                 v
         BI dashboard or app
```

### Project Features

- [ ] Batch ingestion
- [ ] Semi-structured JSON processing
- [ ] Incremental transformation pipeline
- [ ] Star schema
- [ ] Slowly changing dimension Type 2
- [ ] Data-quality tests
- [ ] Role-based access control
- [ ] Dynamic masking and row-level security
- [ ] Workload-isolated warehouses
- [ ] Monitoring and alerts
- [ ] Query-performance optimization
- [ ] Cost monitoring and controls
- [ ] Time Travel and zero-copy clones
- [ ] Git-based development
- [ ] dbt or SQL-based transformations
- [ ] CI/CD deployment
- [ ] Backup and disaster-recovery design
- [ ] Documentation and production runbook

## Recommended Learning Order

1. Phases 1-3: Fundamentals, architecture, and objects
2. Phases 4-6: Data formats, loading, and pipelines
3. Phase 7: Data modeling
4. Phases 8-9: Warehouses and performance
5. Phases 10-12: Security, governance, and recovery
6. Phases 14-15: Cost and production monitoring
7. Phases 16 and 19-20: Integration, DevOps, and testing
8. Phases 13, 17, and 18: Sharing, Snowpark, and AI specialization

## Learning Rule

Do not try to memorize every command. First understand what problem a feature solves, then practice it, and finally learn how to operate it safely in production.
