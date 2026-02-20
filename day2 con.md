
---

# 🏦 Unified Payment Processing Engine (UPPE)

## 1️⃣ Background Story – Why This System Exists

### Organization Context

A mid-to-large Indian bank is modernizing its **real-time payment processing platform**.

The bank handles:

* UPI transfers
* IMPS transfers
* NEFT (near real-time)
* Internal account transfers
* Merchant settlements

Current pain points:

* Fraud detection system latency
* Occasional dependency outages
* Increasing TPS (Transactions Per Second)
* Settlement processing delays
* Retry chaos during outages

The bank decides to build:

> A High-Throughput, Concurrent, Fault-Tolerant Payment Processing Engine

---

# 🎯 Business Goals

1. Process 10,000+ TPS during peak hours.
2. Guarantee no double debit.
3. Ensure deterministic outcomes.
4. Protect downstream fraud and balance systems.
5. Provide strong auditability (regulatory compliant).
6. Handle failures without losing transactions.
7. Support end-of-day settlement in parallel.

---

# 💰 Monetary Standard (Banking Compliance)

All amounts are stored in **minor units (long)**.

Example:

| Display Value | Stored Value |
| ------------- | ------------ |
| ₹100.25       | 10025        |
| ₹1.00         | 100          |

Reasons:

* No floating-point errors
* Deterministic arithmetic
* Faster than BigDecimal under heavy load
* Regulatory compliance requirement

---

# 🧠 Core Responsibilities of the Engine

For each transaction:

1. Accept instruction
2. Validate request
3. Check sufficient balance
4. Call Fraud system
5. Calculate risk score
6. Make decision
7. Persist to ledger (idempotent)
8. Audit everything
9. Retry transient failures
10. Support aggregation for settlement

---

# 🏗 High-Level Architecture (Conceptual)

```
Client → Transaction Queue
              ↓
       Core Worker Pool
              ↓
    ┌─────────────────────┐
    │   Processing Flow   │
    │ Validation          │
    │ Fraud (Async)       │
    │ Balance             │
    │ Risk                │
    │ Decision            │
    │ Ledger              │
    └─────────────────────┘
              ↓
      Retry Scheduler
              ↓
     Settlement Engine
```

---

# ⚠ Key Technical Challenges

| Challenge         | Why It Matters           |
| ----------------- | ------------------------ |
| Fraud latency     | 200ms–2s per call        |
| Fraud outages     | External vendor          |
| Balance DB spikes | High read load           |
| Thread starvation | Too many blocking calls  |
| Double debit      | Catastrophic failure     |
| Retry storms      | Can overload system      |
| Settlement delay  | Impacts merchant payouts |

---

# 🧭 Incremental Phase Roadmap

We will build the system in 11 phases.

Each phase solves a real production problem.

---

# 📌 Complete Phase List

---

## 🔹 Phase 1 — Sequential Processing Baseline

* Single-thread pipeline
* Blocking fraud
* No concurrency
* Deterministic correctness baseline

Goal:
Understand full flow before optimization.

---

## 🔹 Phase 2 — Thread Pool (ExecutorService)

* Fixed-size worker pool
* Multiple transactions processed concurrently
* Controlled thread management

Goal:
Remove single-thread bottleneck.

---

## 🔹 Phase 3 — Concurrent Validation

* Independent validation rules executed in parallel
* Validation time reduced

Goal:
Optimize CPU-bound logic.

---

## 🔹 Phase 4 — Asynchronous Fraud (CompletableFuture)

* Fraud executed asynchronously
* Dedicated fraud executor
* Core worker not blocked unnecessarily

Goal:
Improve latency and throughput.

---

## 🔹 Phase 5 — Async Orchestration (thenApply vs thenCompose)

* Clean async chaining
* Proper flattening of async steps
* Avoid nested futures

Goal:
Maintain readable and maintainable async pipeline.

---

## 🔹 Phase 6 — Parallel Merge (thenCombine, allOf)

* Balance + Risk executed in parallel
* Combine results deterministically
* Support batch handling

Goal:
Reduce end-to-end latency.

---

## 🔹 Phase 7 — Timeout Enforcement

* Fraud SLA boundary defined
* Timeout fallback policy
* Late result discard logic

Goal:
Prevent infinite dependency waiting.

---

## 🔹 Phase 8 — Rate Limiting (Semaphore / Bulkhead)

* Limit concurrent fraud calls
* Protect downstream systems
* Introduce backpressure

Goal:
Prevent cascading failure.

---

## 🔹 Phase 9 — Comprehensive Error Handling

* Deterministic failure mapping
* Exception recovery
* Retry classification
* Audit on failure

Goal:
No undefined state ever.

---

## 🔹 Phase 10 — Scheduled Retry (ScheduledExecutorService)

* Retry transient failures
* Isolated retry executor
* Idempotency enforcement

Goal:
Operational resilience.

---

## 🔹 Phase 11 — ForkJoin Aggregation (Settlement)

* Parallel aggregation of ledger entries
* Separate compute pool
* Read-only dataset processing

Goal:
Efficient end-of-day settlement.

---

# 🛡 Banking Design Principles Applied Across All Phases

1. Immutable transaction object
2. Minor-unit money modeling
3. Idempotency key enforcement
4. Strict state transitions
5. Dedicated executor per responsibility
6. No shared mutable state
7. Terminal states are final
8. All decisions audited
9. Thread isolation
10. Graceful degradation under load

---

# 🧩 State Model (Preview)

Every transaction will eventually reach:

```
APPROVED
DECLINED
MANUAL_REVIEW
RETRY_PENDING
```

No transaction may remain in:

* PROCESSING forever
* UNKNOWN state

---



