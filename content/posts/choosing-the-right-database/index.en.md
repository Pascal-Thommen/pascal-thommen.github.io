---
title: "Choosing the Right Database: PostgreSQL as the Architectural Standard"
date: 2026-10-09T10:00:00Z
description: "Why PostgreSQL is the default architecture for modern projects, how it replaces niche databases, and the few scenarios where specialized engines are actually justified."
summary: "PostgreSQL is the default answer for modern software projects. This guide evaluates functional and operational criteria, highlighting when deviations actually make sense."
tags: ["Databases", "PostgreSQL", "Software Architecture", "Business Informatics"]
categories: ["Enterprise Systems", "Architecture"]
author: "Pascal Thommen"
hidemeta: false
ShowReadingTime: false
ShowBreadCrumbs: true
draft: false
---

Which database fits a new software project? In most cases, PostgreSQL: including documents, geospatial data, full-text search, and AI embeddings. Anything else requires strong justification.

---

## Evaluation Criteria

Every database must be evaluated along two dimensions:

1. **Functional Requirements (FR):** What can the system deliver technically (ACID transactions, relations, flexible querying)?
2. **Non-Functional Requirements (NFR):** What are the operational, maintenance, staffing, and licensing costs, and how well does the system protect against vendor lock-in?

Functional requirements are quickly satisfied by many systems today. Long-term economic viability, however, is determined by non-functional requirements. Maintainability, low Total Cost of Ownership (TCO), and architectural independence always take precedence over short-lived hype features.

The architectural principle "Choose Boring Technology" (coined at Etsy) captures this perfectly: every team has a small budget of "innovation tokens", typically no more than three. Spending these tokens on experimental database engines leaves fewer resources for the core business. The rule: choose the most boring technology that reliably solves the problem.

---

## Why PostgreSQL Is the Standard

PostgreSQL forms the foundation of modern software architecture. Its modular design natively covers requirements that previously demanded dedicated niche databases:

<details open>
<summary>Niche systems replaced by PostgreSQL</summary>

* **Document Databases (e.g., MongoDB):** PostgreSQL stores JSONB in binary format and queries nested attributes rapidly using GIN indexes.
* **Vector Databases (e.g., Pinecone):** Executes high-dimensional vector searches with `pgvector` and HNSW indexes directly via SQL.
* **Geospatial Databases:** Sets the global industry standard for spatial queries with `PostGIS`.
* **Time-Series & IoT Databases (e.g., InfluxDB):** Ingests millions of sensor metrics efficiently using the `TimescaleDB` extension or native table partitioning.
* **Graph Databases:** Resolves shallow relationship trees, org charts, or bills of materials using recursive CTEs in standard SQL, without a graph engine.
* **Full-Text Search Engines:** Provides built-in linguistic analysis for standard search queries without external search clusters.

</details>

In addition, PostgreSQL offers complete licensing security. It runs under a truly free open-source license without a controlling corporation, without audit traps, and without restrictive licenses like the SSPL. SQL talent is globally available.

---

## The Hidden Costs of Specialized Databases

Introducing multiple specialized databases in parallel incurs massive operational overhead:

* **The Dual-Write Catastrophe:** If customer records live in PostgreSQL, product catalogs in MongoDB, and the search index in Elasticsearch, every application write must hit all three systems simultaneously. If the third write fails, data stores diverge. Fixing asynchronous data states after the fact is exceedingly expensive.
* **Day-2 Operations:** Starting a database container takes five minutes (Day 1). Real complexity begins on Day 2: How do zero-downtime backups work? Does the engine support Point-in-Time Recovery (PITR) to recover from operational mistakes down to the second? How smooth are major version upgrades and security patches? PostgreSQL has offered standardized, battle-tested tooling for decades. For niche databases, teams must often build their own operational tooling.

---

## The Exceptions: When to Pick Another System

A strict distinction must be drawn between **standalone primary databases** (which replace PostgreSQL) and **secondary helper databases** (which offload PostgreSQL).

### Part 1: Standalone Primary Databases

Here, another database acts as the single source of truth for regular projects. Extreme edge cases and hyperscale scenarios follow in Part 3.

#### 1. File-Based SQL (SQLite)

When server infrastructure does not exist or is intentionally avoided: mobile apps, desktop software, edge hardware, and CLI tools.

SQLite stores the entire database in a single local file. It requires no server daemon, incurs zero operating costs, and runs completely maintenance-free. Its boundary is architectural: SQLite permits only one writing process at any given moment. Once multiple users write concurrently across a network, it is ruled out.

#### 2. The Hosting Exception: MariaDB

PostgreSQL consumes slightly more idle memory than MariaDB. At current cloud prices (a 4 GB RAM server on Hetzner costs around 6 Euros per month), this difference amounts to a few cents. Sacrificing features and reliability for negligible RAM savings is an unjustified downgrade for custom software development.

MariaDB remains justified when external constraints dictate it: legacy CMS software like WordPress, rigid shared hosting packages, or operations teams already standardized on it. MySQL is disqualified due to its ownership by Oracle, leaving MariaDB as the open alternative.

---

### Part 2: Secondary Databases for Offloading PostgreSQL

These systems never replace PostgreSQL. They operate alongside it as specialized accelerators. There is no risk of inconsistent business data: PostgreSQL remains the single source of truth, while the secondary system handles throughput.

#### 3. In-Memory Cache (Valkey, Redis)

When read traffic demands response times under one millisecond.

The combination of **PostgreSQL as the persistent primary store** and **Valkey or Redis as an in-memory database** is the global industry standard for high-throughput applications. The application reads from fast RAM (cache-aside pattern) and queries PostgreSQL only on cache misses. Cache entries are invalidated deliberately upon data changes.

Example: A product page receives 5,000 requests per second, while catalog data updates once per hour. Valkey delivers responses in microseconds directly from memory, bypassing disk I/O and SQL parsing entirely.

Valkey (hosted by the Linux Foundation) is the license-safe choice for new projects following Redis's shift to restrictive licensing in 2024.

#### 4. Columnar Analytics / OLAP (ClickHouse, DuckDB)

When analytical aggregations across tens of millions of rows degrade transactional performance.

Columnar databases cannot replace transactional engines: they are optimized for append-only batch ingestion and perform poorly on row-level updates. They serve strictly as analytics warehouses populated asynchronously (via scheduled ETL batches or event streaming) from PostgreSQL.

Example: An analysis summarizing 50 million sales records by month and region. As a row-oriented database, PostgreSQL reads every row with all columns from disk. A columnar engine like ClickHouse reads solely the required metric columns, scanning billions of data points in seconds. DuckDB provides this exact advantage in an embedded, serverless format within the application process.

---

### Part 3: Extreme Edge Cases and Hyperscale

Adopting standalone niche specialists is only justified at extreme physical limits:

* **Massive IoT Write Throughput (Apache Cassandra, ScyllaDB):** Consistently exceeding 100,000 writes per second from global sensor fleets. The price paid is eventual consistency and heavy administrative overhead. For regular IoT workloads, PostgreSQL with TimescaleDB handles the volume cleanly.
* **Horizontal Multi-Node Sharding (Distributed SQL like CockroachDB, TiDB):** PostgreSQL scales comfortably vertically (larger CPU/RAM servers) and horizontally via read replicas for high availability and read distribution for 99.9 percent of companies. True horizontal write sharding across dozens of worldwide nodes is predominantly required by hyperscalers like Uber, Netflix, or Amazon.
* **Deep Network Traversal (Neo4j):** Only required for graph path analysis extending beyond three hops (fraud detection, anti-money laundering). Flat relationships are handled cleanly in PostgreSQL.
* **Large-Scale AI Platforms (Qdrant, Milvus):** Only necessary once vector volumes exceed 20 to 50 million high-dimensional embeddings.
* **Proprietary Legacy Systems (Oracle Database, IBM DB2):** Historical technical debt. Commercially and architecturally unjustified for new projects due to exorbitant licensing fees, audit liabilities, and vendor lock-in.

---

## Summary

1. **PostgreSQL is the architectural default:** Any deviation must be justified by hard technical boundaries or organizational mandates, not subjective preference or hype.
2. **Helper systems offload, specialists add liabilities:** Combining PostgreSQL with an in-memory cache (Valkey) solves nearly all scaling challenges. Specialized engines should only be introduced when the primary engine measurably reaches its physical limits.
3. **Operational costs outweigh license costs:** Every added database requires dedicated monitoring, backup routines, and patching processes. PostgreSQL addresses the vast majority of software requirements under a single, manageable operational model.
