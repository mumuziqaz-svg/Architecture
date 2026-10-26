# Software Requirements Specification (SRS)
## Enhanced Real-Time Risk Control Engine for Bank Transfer Systems
**Based on Baseline Architecture Analysis of Open-Source Project "Radar"**

* **Document Version:** 1.0.0
* **Standard Compliance:** IEEE 830 / ISO/IEC/IEEE 29148 Standard
* **Date:** October 2026
* **Status:** Proposed Architecture Specification

---

## 1. Introduction

### 1.1 Purpose
This Software Requirements Specification (SRS) defines the functional and non-functional requirements for refactoring and enhancing the open-source real-time risk control engine **Radar** (`com.pgmmers:radar`). The objective is to bridge the architectural gap between a lightweight standalone decision engine and the stringent latency, throughput, and availability demands of high-concurrency commercial bank transfer pipelines.

### 1.2 Document Conventions
This document adheres to the IEEE standard terminology:
* **SHALL**: Indicates a mandatory requirement.
* **SHOULD**: Indicates a recommended requirement that is subject to trade-off evaluation.
* **MAY**: Indicates an optional design capability.

### 1.3 Intended Audience
* **Software Architects & Reviewers**: To evaluate architectural trade-offs, pattern applications, and resilience strategies.
* **Backend Engineers**: To implement pipeline filters, local caching tiers, circuit breakers, and asynchronous event pipelines.
* **Risk Control Operations & Compliance Staff**: To understand rule evaluation pipelines, fail-safe modes, and audit logging.

### 1.4 Project Scope
The target system serves as the synchronous transaction evaluation gatekeeper and asynchronous behavioral monitor for an online banking transfer subsystem. It ingests transactional events, computes dynamic features (sliding-window metrics, IP-derived location, device data), evaluates business-configured risk rules via dynamic scripting (Groovy), invokes machine learning scores (TensorFlow), issues an execution verdict (`PASS`, `CHALLENGE`, `REJECT`), and asynchronously commits immutable audit trails.

### 1.5 References
1. Mark Richards, *Software Architecture Patterns*, O'Reilly Media.
2. Michael Nygard, *Architecture Decision Records (ADR)*.
3. Open-source repository: `wfh45678/radar` (Spring Boot 2.2.5, Redis, Groovy, Elasticsearch, MySQL).

---

## 2. Overall Description

### 2.1 Product Perspective
The baseline Radar system operates as a layered monolithic/service architecture where an upstream core banking transfer service synchronously waits for decision results over HTTP/REST. 

Under the enhanced architecture, the system operates across three tiers:
1. **Business Ingestion Tier**: Core banking transfer service protected by a client-side circuit breaker.
2. **Radar Decision Engine Core**: Two-level pipeline combining L1 local in-memory pre-filtering (Pipes & Filters) and L2 complex aggregation.
3. **Storage & Asynchronous Infrastructure Tier**: Redis for shared hot counters, Caffeine for node-local state, MySQL for rule configuration metadata, and Kafka + Elasticsearch for asynchronous deep audit and behavioral analytics.

```
[Core Banking Service]
        │
   (HTTP/RPC + Circuit Breaker)
        ▼
[Radar Engine: L1 In-Memory Filter (Caffeine)] ──Hit Hard Limit/Blacklist──► [REJECT (<1ms)]
        │ (Passed 80% Traffic)
        ▼
[Radar Engine: L2 Aggregation Engine (Redis)]  ──Timeout / Error──────────► [Fallback Tier]
        │
   (Synchronous Verdict: PASS / CHALLENGE)
        │
        ├─────────────────────────────────────────┐
        ▼                                         ▼
[Execute Transfer]                    [Kafka Asynchronous Event Bus]
                                                  │
                                                  ▼
                                      [Worker / Elasticsearch Audit]
```

### 2.2 User Characteristics
* **End Customers**: Require transparent, instant transfers with response times strictly under 200ms end-to-end.
* **Risk Strategists**: Configure and modify fraud rules dynamically via `radar-admin` without requiring service redeployment.
* **System Auditors**: Require unalterable historical decision records for anti-money laundering (AML) regulatory compliance.

### 2.3 Operating Environment
* **Runtime**: Java Virtual Machine (JDK 1.8 / 11+ compatible).
* **Framework**: Spring Boot 2.2.5.RELEASE.
* **Data Stores**: Redis Cluster 6.x+, MySQL 8.0+, Elasticsearch 7.11.2+.
* **Messaging**: Apache Kafka 2.x+ / RabbitMQ.

### 2.4 Design and Implementation Constraints
* **CON-1 (Backward Compatibility)**: Existing rule definitions written in Groovy 2.4.6 and MySQL configuration schemas SHALL remain compatible.
* **CON-2 (Lightweight Footprint)**: Refactoring SHALL prioritize low-overhead in-process libraries (e.g., Caffeine, Resilience4j) over heavyweight distributed streaming clusters (e.g., full Apache Flink cluster).
* **CON-3 (Security & Compliance)**: Customer PII (Personally Identifiable Information) and bank account numbers SHALL be masked in all Redis keys and audit log streams.

---

## 3. Specific Requirements

### 3.1 External Interface Requirements

#### 3.1.1 Software Interfaces
* **SI-1 (Core Ingress API)**:
  * Protocol: Synchronous HTTP/1.1 POST / gRPC.
  * Endpoint: `/api/v1/risk/evaluate`
  * Input Payload: JSON containing `transactionId`, `sourceAccount`, `targetAccount`, `amount`, `currency`, `timestamp`, `clientIp`, `deviceId`.
  * Response Payload: JSON containing `transactionId`, `verdict` (`PASS`, `CHALLENGE`, `REJECT`), `score`, `riskRuleHits`, `executionTimeMs`.
* **SI-2 (Asynchronous Event Egress)**:
  * Protocol: Apache Kafka Topic `risk.transfer.events.v1`.
  * Payload: Protobuf/JSON schema carrying decision context, raw request attributes, evaluated features, and verdict timestamp.

---

### 3.2 Functional Requirements

#### 3.2.1 Level-1 In-Memory Pre-Filtering (Pipes & Filters)
* **FR-1.1**: The system SHALL maintain a synchronized local memory cache (L1 Cache via Caffeine) containing static high-risk account blacklists and system-level hard limits (e.g., maximum single-transaction amount).
* **FR-1.2**: If a transaction violates a hard limit or matches a local blacklist entity, the engine SHALL immediately terminate evaluation, return a `REJECT` verdict, and short-circuit any subsequent remote storage I/O.
* **FR-1.3**: The L1 cache SHALL support dynamic invalidation within 5 seconds upon rule or list modification in `radar-admin`.

#### 3.2.2 Level-2 Dynamic Feature & Rule Aggregation
* **FR-2.1**: For requests passing L1, the engine SHALL compute dynamic sliding-window features (e.g., transfer count in past 10 minutes, total sum across 24 hours) against Redis.
* **FR-2.2**: Multi-feature Redis operations SHALL be dispatched using Redis Pipelining or batched `MGET` to constrain network round-trip time (RTT).
* **FR-2.3**: Dynamic business rules defined by risk analysts SHALL be evaluated through pre-compiled Groovy scripts without runtime compilation overhead on every transaction.

#### 3.2.3 Tiered Decision Enforcement
* **FR-3.1**: The evaluation response SHALL support a tri-state verdict model:
  1. `PASS`: Transaction allowed to proceed directly.
  2. `CHALLENGE`: Transaction held pending secondary out-of-band verification (SMS OTP / Biometric authentication).
  3. `REJECT`: Transaction blocked immediately due to confirmed fraudulent patterns.

#### 3.2.4 Asynchronous Audit and AML Archival
* **FR-4.1**: Following synchronous verdict emission, the engine SHALL emit full transaction event data to an asynchronous message broker (Kafka).
* **FR-4.2**: Decoupled consumer workers SHALL consume the message topic to perform asynchronous Elasticsearch indexing and long-window behavioral correlation without blocking upstream transaction threads.

---

### 3.3 Non-Functional Requirements (Quality Attributes)

#### 3.3.1 Performance & Latency (NFR-1)
* **NFR-1.1 (Latency)**: Under a baseline load of 2,000 Transactions Per Second (TPS), the synchronous 99th percentile (P99) risk evaluation latency SHALL be $\le 50\text{ ms}$.
* **NFR-1.2 (L1 Fast-Path Latency)**: Hard-limit and blacklist evaluations executed solely within the L1 filter SHALL complete in $\le 1\text{ ms}$.
* **NFR-1.3 (Throughput)**: A single enhanced 4-core 8GB container node SHALL sustain at least 1,500 TPS without exceeding 80% sustained CPU utilization.

#### 3.3.2 Availability & Resilience (NFR-2)
* **NFR-2.1 (Uptime Target)**: The bank transfer verification capability SHALL achieve an availability of $99.999\%$ (no more than 5.26 minutes of downtime per year).
* **NFR-2.2 (Circuit Breaking)**: Upstream clients SHALL wrap risk check invocations with a Circuit Breaker configured with a 50ms execution timeout and a 50% failure rate trip threshold over a sliding window of 100 calls.
* **NFR-2.3 (Tiered Fallback)**:
  * When the circuit breaker is in the `OPEN` state:
    * Low-risk transactions ($\le \$1,000$ USD equivalent to recognized payees) SHALL execute a **Fail-Open** policy (pass transaction, tag for post-audit).
    * High-risk or unfamiliar payee transactions SHALL execute a **Challenge-Fallback** policy (require secondary SMS/Biometric validation).
  * System-wide hard crashes SHALL NOT cause unhandled 500 errors to interrupt core banking ledger execution.

#### 3.3.3 Scalability (NFR-3)
* **NFR-3.1**: The decision engine tier SHALL remain stateless, allowing linear horizontal scaling by adding engine container instances behind an internal load balancer.
* **NFR-3.2**: Redis feature storage SHALL support cluster mode partitioning by hashing on `sourceAccount` to avoid hotspot keys during payroll batch events.

#### 3.3.4 Security & Auditability (NFR-4)
* **NFR-4.1**: Every completed risk assessment SHALL generate an immutable trace containing a cryptographically verifiable transaction UUID, rule version identifier, and feature snapshot.
* **NFR-4.2**: Audit logs stored in Elasticsearch SHALL be retained for a minimum of 180 days to fulfill banking financial supervisory requirements.