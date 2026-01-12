# High-Concurrency Flash Sale System

> **A robust backend inventory engine designed to handle massive concurrent traffic spikes (e.g., "Black Friday" sales) without data corruption or overselling.**

---

## 📌 The Challenge

In a real-world flash sale, thousands of users try to buy a limited number of items at the exact same second.
Most standard web applications fail under this load due to **race conditions**.

* **The Problem:** If User A and User B read `Stock = 1` simultaneously, both buy it, and the stock becomes `-1`.
* **The Goal:** Build a system that guarantees **100% data consistency** (zero overselling) while maintaining **high throughput**.

---

## 🏗️ Architecture Evolution

This project was not just *built*; it was **evolved** through three distinct phases, each addressing a specific engineering problem.

---

### Phase 1: The Naive Implementation (The "Bug")

* **Approach:** Standard Java flow — `Read DB` → `If Stock > 0` → `Decrement` → `Save`.
* **Result:** Catastrophic failure under load. During testing, **898 items were oversold** because threads overwrote each other’s updates.

```text
+----------------+    +----------------+
|  User A (T1)   |    |  User B (T2)   | (Simultaneous Requests)
+-------+--------+    +-------+--------+
        |                     |
        v                     v
+--------------------------------------+
|       Spring Boot Application        |
+-------+---------------------+--------+
        | 1. Read Stock=100   | 1. Read Stock=100
        v                     v
+--------------------------------------+
|        PostgreSQL Database           |
+--------------------------------------+
        ^                     ^
        | 2. Write Stock=99   | 2. Write Stock=99

RESULT: Two items sold, but DB stock only decreased by 1.
```

---

### Phase 2: Pessimistic Locking (The "Safe" Way)

* **Approach:** Used PostgreSQL `SELECT ... FOR UPDATE` to lock the inventory row.
* **Result:** **100% accuracy**. No overselling.
* **Trade-off:** **High latency**. The database became a bottleneck as thousands of users waited in a single-file queue.

```text
[ DATABASE TRANSACTION QUEUE ]
+-----------------------------+------------------------------+
|  User A (Processing...)     |  User B (WAITING...)         |
+-----------------------------+------------------------------+
              |
              | 2. ACQUIRE LOCK (PESSIMISTIC_WRITE)
              v
+-----------------------------+
|    PostgreSQL Database      |
| (Row is locked by User A)   |
+-----------------------------+
```

---

### Phase 3: Hybrid Architecture (The "Senior" Solution)

* **Approach:** Combined **Redis atomic counters** with **database-level locking**.
* **How it works:** Redis acts as a high-speed *bouncer*. It rejects invalid requests in memory (microseconds) before they ever reach the database. Only the "winning" requests proceed to PostgreSQL for final persistence.
* **Result:** **High speed + 100% accuracy**.

```text
+--------------------------------------------------+
|            Massive Concurrent Traffic            |
+------------------------+-------------------------+
                         |
                         v
+--------------------------------------------------+
|             Spring Boot Application              |
+------------------------+-------------------------+
                         | 1. ATOMIC DECREMENT
                         v
          +---------------------------+
          |    Redis Cache (NoSQL)    | <-- The "Bouncer"
          | (In-Memory, Single-Thread)|
          +-------------+-------------+
                        |
      (90% of traffic)  |  (10% "Winners")
      +<---[ FAIL < 0 ]-+-[ PASS >= 0 ]--->+
      |                                    |
      v                                    v
+----------------+       +----------------------------------+
| Fast "Sold Out"|       |   PostgreSQL Database (The Vault)|
| Response       |       | 2. Acquire Pessimistic Lock      |
+----------------+       | 3. Safely Update Stock           |
                         +----------------------------------+
```

---

## 🧰 Tech Stack

* **Language:** Java 25
* **Framework:** Spring Boot 5.8.9 (Web, Data JPA)
* **Database:** PostgreSQL 16
* **Caching / Locking:** Redis 7 (Atomic Counters)
* **Testing:** JUnit 5, ExecutorService (Concurrency Simulation)
* **Infrastructure:** Docker & Docker Compose

---

## 🧪 Testing Concurrency

### What it does

* Spins up **1,000 threads**.
* Hammers the API **simultaneously**.
* Asserts that **exactly 100 items** were sold and **stock is 0**.

If the system were buggy, this test would expose negative stock values or more than 100 successful orders.
