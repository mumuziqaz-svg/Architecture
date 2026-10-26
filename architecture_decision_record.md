# 1. Architectural Restructuring for High-Concurrency Bank Transfer Risk Control

* **Status:** Proposed
* **Date:** 2026-10-26
* **Deciders:** Software Architecture Project Team
* **Target System:** Radar Real-Time Risk Control Engine (`com.pgmmers:radar`)

---

## Context and Problem Statement

The baseline open-source project **Radar** provides a modular real-time risk control solution utilizing Spring Boot, dynamic Groovy rule execution, and centralized storage (Redis, MySQL, Elasticsearch). However, when evaluated against the quality attributes of a commercial online banking transfer subsystem (sub-50ms latency, high concurrent TPS, and 99.999% availability), the existing architecture exhibits three critical deficiencies:

1. **Remote Network I/O Amplification**: Each evaluation synchronously issues multiple sequential network round-trips to Redis to fetch sliding-window counters and query blacklists, driving P99 latency above 100ms under network jitter.
2. **Cascading Failure Vulnerability (Tight Synchronous Coupling)**: The upstream core banking service waits synchronously on Radar over REST HTTP. The absence of fault isolation mechanisms (such as circuit breakers or thread pool bulkheads) risks exhausting upstream connection pools if Radar encounters GC pauses or database latency spikes.
3. **Heavy Synchronous Execution Path**: Hard rejection checks, complex dynamic script interpretation, model scoring, and Elasticsearch audit logging are conflated into a single synchronous thread, limiting vertical scalability and horizontal throughput.

---

## Decision Drivers

* **Performance**: Synchronous P99 latency must not exceed 50ms under peak transfer conditions.
* **Availability**: Failure or latency spikes in the risk engine must not cause outages in the core bank transfer channel.
* **Implementation Feasibility**: The solution must avoid introducing heavy, operationally complex big data streaming frameworks (e.g., multi-node Apache Flink clusters) while delivering measurable architectural resilience.
* **Alignment with Architectural Styles**: Practical application of classic architectural patterns from Mark Richards' *Software Architecture Patterns* (Layered, Pipes & Filters, Event-Driven, Space-Based In-Memory Caching).

---

## Considered Options

* **Option 1: Status Quo (Scale Redis and JVM Vertically)**
  * Retain existing single-path synchronous Groovy + Redis architecture; increase Redis replicas and allocate larger JVM heap space.
* **Option 2: Heavyweight Distributed Stream Processing Overhaul**
  * Completely replace `radar-engine` with an Apache Flink + Kafka + RocksDB stateful stream-processing topology.
* **Option 3: Two-Level Pipeline with Local Memory Filtering, Resilience4j Circuit Breaker, and Asynchronous Event Bypass (Selected)**
  * Restructure the evaluation pipeline using the **Pipes & Filters** pattern with node-local **Caffeine** L1 caching.
  * Encapsulate ingress calls using the **Circuit Breaker & Bulkhead** pattern.
  * Offload audit logging and heavy post-decision calculations to an **Event-Driven Architecture (EDA)** broker topology using Kafka.

---

## Decision Outcome

**Chosen Option: Option 3**.

We decided to refactor the transaction decision path into a hybrid two-level pipeline supported by client-side resilience and an asynchronous event bypass:

### Architectural Components of the Selected Solution

1. **L1 Local Memory Filter (Pipes & Filters Pattern)**:
   * Introduce Caffeine as an in-process L1 cache inside `radar-engine`.
   * Static blacklists, high-risk BIN ranges, and hard single-transaction limit rules are evaluated directly in JVM memory (<1ms).
   * Short-circuiting logic terminates evaluation immediately upon an L1 violation, eliminating unnecessary remote Redis I/O for up to 80% of trivial or explicitly malicious traffic.

2. **Upstream Circuit Breaker & Tiered Fallback (Resilience & Bulkhead Patterns)**:
   * Deploy Resilience4j at the transfer service invocation boundary with a strict 50ms timeout.
   * If the risk engine stalls, the circuit trips (`OPEN`) and activates automated tiered fallbacks:
     * Low-value transfers are handled via **Fail-Open** with an audit flag.
     * High-value transfers trigger a **Challenge Fallback** (prompting the customer for two-factor SMS/biometric verification), preserving core banking service continuity while limiting fraud exposure.

3. **Asynchronous Bypass for Audit & Deep Analytics (Event-Driven Broker Topology)**:
   * Once a synchronous verdict (`PASS` / `CHALLENGE`) is rendered, transaction metadata and feature context are dispatched asynchronously to a Kafka topic.
   * Background workers consume the topic to asynchronously write historical records to Elasticsearch and trigger deep anti-money laundering (AML) pattern matching.

---

## Pros and Cons of the Selected Option

### Positive Consequences
* **Extreme Latency Reduction**: Eliminating Redis network RTTs for L1-evaluated traffic reduces average evaluation latency from >40ms to sub-millisecond ranges. Overall P99 stays comfortably below 30ms.
* **Failure Decoupling**: Core bank transfer operations are insulated from downstream risk engine crashes and network partitions.
* **Throughput Scalability**: Stripping Elasticsearch indexing and deep analytics from the synchronous thread frees CPU and connection resources for primary transaction verification.
* **Cost Efficiency**: Implementation leverages lightweight, in-process Java components without requiring dedicated streaming infrastructure.

### Negative Consequences and Mitigations
* **Cache Invalidation Latency**:
  * *Trade-off*: Local Caffeine instances risk serving stale blacklist data if an update occurs in `radar-admin`.
  * *Mitigation*: Broadcast cache invalidation events via a lightweight Redis Pub/Sub topic to achieve sub-second synchronization across all engine nodes.
* **Eventual Consistency in Audit Logs**:
  * *Trade-off*: Elasticsearch audit records lag synchronous transactions by several hundred milliseconds.
  * *Mitigation*: Acceptable for banking regulatory compliance, as operational transaction ledgers maintain synchronous primary database guarantees.
* **Operational Overhead**:
  * *Trade-off*: Introduces a message broker (Kafka) dependency into the production topology.
  * *Mitigation*: Utilizes standard managed Kafka or lightweight cluster configurations, reusing Radar's native `radar-kafka-demo` module structure.

---

## Architectural Pattern Mapping Summary

| Problem in Baseline Radar | Chosen Pattern / Technique | Component in Refactored Design |
| :--- | :--- | :--- |
| Remote I/O Amplification | **Pipes & Filters** + In-Memory Caching | L1 Filter using Caffeine within `radar-service-impl` |
| Synchronous Cascading Failure | **Circuit Breaker** & **Bulkhead** | Resilience4j wrapper at API Ingress boundary |
| Heavy Real-Time Processing Path | **Event-Driven Architecture (EDA)** | Kafka broker + decoupled async consumers for ES audit |