# Architecture Decision Record (ADR)
## Project: Radar Real-Time Risk Control Engine — High-Concurrency Optimization

---

### Baseline System Context & Problem Statement
- **Baseline Reference**: Radar Open-Source Real-Time Risk Engine (`wfh45678/radar`, GitHub: `https://github.com/wfh45678/radar`)
- **System Positioning & Scope**: Radar is a lightweight, general-purpose real-time anti-fraud decision engine designed for small-to-medium financial technology platforms and early-stage risk control systems. It decouples risk policy management from business applications by evaluating incoming transaction events against dynamically compiled **Groovy** rules. Its baseline architecture comprises two core components: `radar-admin` (rule configuration and management) and `radar-engine` (real-time scoring via `ContextBuilder` and rule executors), backed by **Spring Boot**, **MySQL**, **Redis**, and **MongoDB/Elasticsearch**.
- **Domain Adaptation & Refactoring Drivers**: While effective for general-purpose, moderate-traffic scenarios (such as login verification or marketing anti-cheating), applying Radar's baseline architecture directly to **High-Concurrency Real-Time Bank Fund Transfers** exposes severe quality attribute bottlenecks:
  1. **Ingress Serialization Overhead**: Reliance on verbose HTTP/1.1 + JSON text payloads introduces avoidable network latency at the entry point.
  2. **Network I/O Amplification**: Every transaction requires synchronous network round-trips to centralized Redis/data stores even for static blacklists and simple threshold checks.
  3. **Cascading Failure Risk**: Tight synchronous coupling without timeout boundaries means any latency spike in the risk engine directly paralyzes core banking transfers.
  4. **Critical-Path Logging Contention**: Synchronous audit log persistence blocks real-time `PASS`/`REJECT` responses under peak traffic.
- **Architectural Goal**: The following ADRs document the structural refactorings applied to transform Radar into a low-latency (`P99 < 50ms`), fault-tolerant, high-throughput risk engine tailored for core banking fund transfers.

---

### ADR-001: Replace HTTP/1.1 REST with gRPC (Protobuf + HTTP/2) for Ingress Communication
- **Status**: Accepted
- **Context**: In high-concurrency bank transfer scenarios, the legacy HTTP/1.1 REST interface relies on verbose JSON text serialization and suffers from head-of-line blocking and repeated TCP/header overhead, introducing avoidable network latency at the entry point (`ContextBuilder`).
- **Decision**: Replace internal service-to-service HTTP REST endpoints between the Core Banking System and `radar-engine` with **gRPC** over **HTTP/2**, using **Protocol Buffers (Protobuf)** for binary serialization.
- **Consequences**:
  * **Positive**: Reduces payload size by ~60–80%, accelerates serialization/deserialization, and enables multiplexed concurrent requests over persistent HTTP/2 connections.
  * **Negative**: Requires `.proto` schema governance across Core Banking and Risk teams; binary payloads are not directly human-readable during raw packet inspection.

---

### ADR-002: Introduce Local In-Memory Cache (Caffeine) for Static Rules & Hot Lists
- **Status**: Accepted
- **Context**: The baseline Radar engine queries external **Redis** instances over the network for every incoming transaction. Under high concurrency, repetitive network round-trips for static blacklists and simple threshold checks create network I/O bottlenecks and inflate tail latency.
- **Decision**: Embed an in-process **Caffeine** local memory cache inside `radar-engine` to store deterministic blacklists/whitelists and static Groovy rules that do not depend on historical sliding-window counters.
- **Consequences**:
  * **Positive**: Eliminates network hops (nanosecond-level local access) for obvious high-risk or whitelisted transfers, significantly reducing Redis QPS and overall engine latency.
  * **Negative**: Bounded by JVM heap capacity; requires an event-driven cache invalidation/refresh mechanism when administrators update lists in `radar-admin`.

---

### ADR-003: Implement 50ms Timeout Circuit Breaker and Tiered Degradation
- **Status**: Accepted
- **Context**: The original synchronous coupling between Core Banking and Radar creates a single point of cascading failure: if Redis experiences network jitter or `radar-engine` saturates under load, Core Banking threads block indefinitely, paralyzing all bank transfers.
- **Decision**: Wrap the `radar-engine` invocation boundary with a **Circuit Breaker** configured with a strict **50ms timeout** and a **tiered degradation strategy**:
  1. *Low-value transactions*: **Fail-Open** (auto-approve to maintain payment continuity).
  2. *High-value transactions*: Fallback to **Step-Up Authentication** (SMS OTP / facial recognition).
- **Consequences**:
  * **Positive**: Completely isolates Core Banking from risk engine slowdowns or outages while balancing business continuity (low-value) against financial security (high-value).
  * **Negative**: Introduces minor controlled risk exposure on small-value transfers during active circuit-open windows; requires careful tuning of timeout thresholds and amount tiers.

---

### ADR-004: Decouple Audit Logging from the Critical Path via Kafka to Elasticsearch
- **Status**: Accepted
- **Context**: Legacy Radar synchronously writes detailed execution audit logs directly to **Elasticsearch** before returning a `PASS`/`REJECT` decision. Any disk I/O spike or indexing slowdown in Elasticsearch directly blocks transaction responses.
- **Decision**: Separate the real-time decision path from post-transaction persistence by introducing **Apache Kafka** as an asynchronous buffer. `radar-engine` returns the decision immediately upon rule scoring and publishes the audit record to Kafka, where background consumers batch-index logs into Elasticsearch.
- **Consequences**:
  * **Positive**: Removes Elasticsearch write latency from the critical path, absorbs traffic spikes ("peak shaving"), and guarantees zero log loss if Elasticsearch undergoes maintenance.
  * **Negative**: Introduces eventual consistency (typically < 1–2 seconds delay) before audit logs become searchable in `radar-admin`; adds Kafka cluster operational overhead.

---

### ADR-005: Hybrid Polyglot Persistence Strategy (MySQL + Redis + Elasticsearch)
- **Status**: Accepted
- **Context**: Different data types within a risk engine have conflicting requirements regarding ACID durability, sub-millisecond access, and full-text searchability.
- **Decision**: Adopt a dedicated three-tier storage architecture:
  * **MySQL**: Authoritative relational store for Groovy rule definitions, versioning, and RBAC configurations requiring ACID transactions.
  * **Redis**: Distributed in-memory key-value store for high-frequency real-time sliding-window counters (e.g., 10-minute transfer velocity).
  * **Elasticsearch**: Distributed document search engine for massive-scale historical audit logs and multi-dimensional compliance queries.
- **Consequences**:
  * **Positive**: Each workload runs on a storage engine optimized for its exact access pattern.
  * **Negative**: Requires maintaining three distinct persistence technologies and clear data synchronization pipelines between them.
