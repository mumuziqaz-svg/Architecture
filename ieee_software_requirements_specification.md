# Software Requirements Specification (SRS)
## Project: High-Concurrency Real-Time Risk Control Engine (Radar Architecture Optimization)

### 1. Introduction
#### 1.1 Purpose
This document specifies the functional and non-functional requirements for the optimized **Radar Real-Time Risk Control Engine**, tailored specifically for high-concurrency online bank transfer scenarios. The objective is to eliminate synchronous bottlenecks, minimize network I/O latency, and prevent cascading failures between the Core Banking Transfer System and the risk engine.

#### 1.2 Scope
The system comprises two primary modules:
- **`radar-admin`**: Web-based management portal for configuring dynamic Groovy rules, managing static blacklists/whitelists, and querying historical audit logs.
- **`radar-engine`**: High-throughput execution engine responsible for ingesting transfer requests, evaluating local and dynamic rules, extracting real-time features, and dispatching audit logs asynchronously.

#### 1.3 Baseline System Overview & Refactoring Drivers
- **Open-Source Baseline Reference**: This project evolves from the open-source **Radar Real-Time Anti-Fraud Engine** (`wfh45678/radar`, GitHub: `https://github.com/wfh45678/radar`).
- **Original Positioning & Applicability**: Radar is a lightweight, general-purpose real-time decision engine built on **Spring Boot**, **Redis**, **MongoDB/Elasticsearch**, and **Groovy** dynamic scripting. It is designed to decouple risk rules from business applications across broad scenarios such as payment risk control, account login security, marketing anti-cheating, and pre-loan screening.
- **Architectural Bottlenecks in Bank Transfer Scenarios**: While Radar satisfies moderate-traffic, general-purpose risk evaluation, applying its baseline architecture directly to **high-concurrency real-time bank fund transfers** (intra-bank, inter-bank, and mobile transfers) exposes critical architectural limitations:
  1. *Ingress & Serialization Overhead*: Legacy HTTP/1.1 + JSON interfaces introduce unnecessary payload bloat and latency at entry.
  2. *Synchronous Network Amplification*: Every request performs remote network queries to centralized Redis/data stores even for static blacklists and simple threshold rules.
  3. *Cascading Failure Vulnerability*: Tight synchronous coupling without timeout boundaries or circuit breakers allows risk engine slowdowns to block core banking transfers.
  4. *Critical-Path Logging Contention*: Synchronous persistence of execution audit logs directly degrades real-time decision throughput under traffic spikes.

---

### 2. System Architecture & Data Flow
The risk evaluation pipeline follows a decoupled, fast-path/slow-path architecture:
1. **Ingress & Context Encapsulation**: Core Banking invokes `radar-engine` via **gRPC**. `ContextBuilder` parses the Protobuf payload, validates mandatory fields, and initializes a standardized internal `Context` object.
2. **Local Pre-Filtering (Fast Path)**: `radar-engine` evaluates static hard rules and deterministic blacklists/whitelists cached in **local JVM memory (Caffeine)** before making any external network calls.
3. **Real-Time Feature Extraction**: For transactions passing local pre-filtering, the feature service queries external **Redis** clusters for sliding-window behavioral metrics (e.g., 10-minute transfer frequency) and populates the `Context`.
4. **Dynamic Rule & Model Evaluation**: The decision engine executes **Groovy dynamic rules** (persisted in **MySQL**, hot-reloaded into memory) and optional ML scoring models to calculate a cumulative Risk Score.
5. **Immediate Response & Asynchronous Logging**: `radar-engine` immediately returns the decision (`PASS`, `CHALLENGE`, or `REJECT`) to Core Banking, while publishing full execution traces to **Kafka** for asynchronous ingestion into **Elasticsearch**.

---

### 3. Functional Requirements

#### FR-1: High-Speed RPC Ingress & Context Building
- **FR-1.1**: The system shall expose a **gRPC** service endpoint using Protocol Buffers (Protobuf) to receive synchronous risk evaluation requests from the Core Banking Transfer System.
- **FR-1.2**: `ContextBuilder` shall validate required transaction attributes (source account, target account, amount, timestamp, device ID, IP) and generate a unique `eventId` within a unified internal `Context` object.

#### FR-2: Local In-Memory Pre-Filtering
- **FR-2.1**: The engine shall maintain an in-process local cache (**Caffeine**) storing deterministic blacklists, trusted whitelists, and static threshold rules (e.g., single-transaction hard limits).
- **FR-2.2**: Transactions matching local hard-reject rules or trusted whitelists shall short-circuit the pipeline immediately, bypassing remote Redis feature queries.

#### FR-3: Dynamic Rule Management & Evaluation
- **FR-3.1**: Administrators shall create, update, and version **Groovy** rule scripts via `radar-admin`, persisted in **MySQL** with full ACID transaction guarantees.
- **FR-3.2**: `radar-engine` shall support zero-downtime hot-reloading of updated Groovy scripts into memory.
- **FR-3.3**: The engine shall aggregate risk scores across all triggered rules and map the total score to configurable thresholds (`PASS`, `CHALLENGE`, `REJECT`).

#### FR-4: Circuit Breaking & Tiered Graceful Degradation
- **FR-4.1**: The Core Banking client boundary shall enforce a strict **50ms timeout** and an automated **Circuit Breaker** when invoking `radar-engine`.
- **FR-4.2**: Upon timeout or circuit-open state, the system shall execute tiered fallback logic based on transaction amount:
  * **Low-Value Transfers** (below configurable threshold): Default to **Fail-Open (`PASS`)** to preserve user experience and payment availability.
  * **High-Value Transfers** (at or above threshold): Downgrade to **Secondary Verification (`CHALLENGE`)** via SMS OTP or biometric facial recognition.

#### FR-5: Asynchronous Audit Logging & Search
- **FR-5.1**: Upon decision completion, `radar-engine` shall publish complete audit payloads (input context, triggered rules, individual rule scores, final decision) to a **Kafka** topic without blocking the gRPC response thread.
- **FR-5.2**: Background Kafka consumers shall consume audit events and index them into **Elasticsearch** to support sub-second regulatory compliance queries and customer dispute investigations.

---

### 4. Non-Functional Requirements

| Category | Requirement Specification |
| :--- | :--- |
| **Latency** | P99 end-to-end risk evaluation latency shall remain **under 30ms** under normal load; hard timeout enforced at **50ms**. |
| **Throughput** | Support **≥ 10,000 TPS** peak concurrency during bank promotional events or payroll cycles. |
| **Availability** | Core transfer path availability of **99.999%** guaranteed via Circuit Breaker isolation and Fail-Open/Challenge degradation. |
| **Data Durability** | Zero loss of regulatory audit logs guaranteed via Kafka disk persistence and consumer offset commits prior to Elasticsearch indexing. |
| **Maintainability** | Rule updates via Groovy scripts must take effect within **5 seconds** without service restarts. |
