# 🏗️ System Design — High-Level Design (HLD) Master Notes

> **Everything you need** to go from zero → interview-ready → production-ready.
> Built from the roadmap, deep-dive articles, and lecture notes — all in one place.

---

## 📚 Table of Contents

| # | Topic |
|---|-------|
| 1 | [Foundations & Core Concepts](#1️⃣-foundations--core-concepts) |
| 2 | [Networking Essentials](#2️⃣-networking-essentials) |
| 3 | [Databases & Storage Systems](#3️⃣-databases--storage-systems) |
| 4 | [Caching](#4️⃣-caching) |
| 5 | [Message Queues & Event Systems](#5️⃣-message-queues--event-systems) |
| 6 | [Load Balancing](#6️⃣-load-balancing) |
| 7 | [Rate Limiting & Throttling ⭐ Deep Dive](#7️⃣-rate-limiting--throttling-deep-dive) |
| 8 | [Distributed System Concepts](#8️⃣-distributed-system-concepts) |
| 9 | [HLD Architecture Patterns](#9️⃣-hld-architecture-patterns) |
| 10 | [Low-Level Design (LLD)](#-low-level-design-lld) |
| 11 | [Scalability & Deployment](#-scalability--deployment) |
| 12 | [Monitoring & Observability](#1️⃣1️⃣-monitoring--observability) |
| 13 | [Interview Questions](#1️⃣2️⃣-system-design-interview-questions) |
| 14 | [1-Month Study Plan](#-1-month-study-plan) |

---

## 1️⃣ Foundations & Core Concepts

> These are the **non-negotiables**. You cannot skip these. Every system design answer starts here.

### 🔹 What is System Design?
System design is the process of defining the **architecture, components, modules, interfaces, and data** for a system to satisfy specified requirements. In interviews, it tests whether you can build something that works **at scale** — handling millions of users, requests, and terabytes of data.

---

### 🔹 Monolithic vs Microservices Architecture

| Feature | Monolithic | Microservices |
|---------|------------|---------------|
| **Structure** | Single deployable unit | Many independent services |
| **Scaling** | Scale the whole app | Scale individual services |
| **Failure** | One bug can crash everything | Isolated failures |
| **Deployment** | Simple but risky | Complex but safe |
| **Best for** | Small teams, early-stage apps | Large teams, high scale |

```
MONOLITH                        MICROSERVICES
┌────────────────────┐          ┌───────┐  ┌───────┐  ┌───────┐
│  UI + Business     │          │  Auth │  │ Order │  │  Pay  │
│  Logic + DB Layer  │   vs     │  Svc  │  │  Svc  │  │  Svc  │
│  (all-in-one)      │          └───┬───┘  └───┬───┘  └───┬───┘
└────────────────────┘              └──────────┴──────────┘
                                           API Gateway
```

---

### 🔹 Scalability — Horizontal vs Vertical

| Type | What it means | Example |
|------|---------------|---------|
| **Vertical Scaling** | Add more power to ONE machine (bigger CPU, RAM) | Upgrade server from 8GB → 64GB RAM |
| **Horizontal Scaling** | Add MORE machines | Go from 1 server → 10 servers behind a load balancer |

> 💡 **Rule of thumb**: Vertical scaling has a ceiling. Horizontal scaling is theoretically infinite — but adds complexity.

---

### 🔹 Latency vs Throughput

- **Latency** = time for **one request** to complete (milliseconds). Lower is better.
- **Throughput** = number of requests the system can handle **per second** (RPS/QPS). Higher is better.

> They are often in tension — optimizing for one can hurt the other.

---

### 🔹 Availability & Reliability

- **Availability** = % of time the system is **up and running**
  - 99.9% = "three nines" → ~8.7 hours downtime/year
  - 99.99% = "four nines" → ~52 minutes downtime/year
- **Reliability** = system **consistently does what it's supposed to do**, even under failure conditions

---

### 🔹 CAP Theorem

> In a distributed system, you can only guarantee **2 out of 3**:

| Property | Meaning |
|----------|---------|
| **C**onsistency | Every read gets the most recent write |
| **A**vailability | Every request gets a (non-error) response |
| **P**artition Tolerance | System works even if network partition happens |

```
        C
       / \
      /   \
     /     \
    A ───── P

Choose any 2:
CA → Traditional SQL DBs (MySQL)
CP → MongoDB, HBase (sacrifice availability)
AP → Cassandra, DynamoDB (sacrifice consistency)
```

---

### 🔹 ACID vs BASE

| | ACID (SQL) | BASE (NoSQL) |
|-|------------|--------------|
| **A** | Atomicity — all or nothing | **B**asically Available |
| **C** | Consistency — valid state always | **S**oft state |
| **I** | Isolation — no interference | **E**ventually consistent |
| **D** | Durability — survives crashes | |
| **Best for** | Banking, payments | Social feeds, analytics |

---

### 🔹 Read-Heavy vs Write-Heavy Systems

| System Type | Examples | Design Focus |
|-------------|----------|--------------|
| **Read-heavy** | Twitter timeline, YouTube | Caching, CDN, read replicas |
| **Write-heavy** | Logging, IoT sensors | Write buffers, Kafka, sharding |

---

## 2️⃣ Networking Essentials

> You can't design a system without knowing how data moves.

### 🔹 HTTP vs HTTPS
- **HTTP** = plain text, no encryption. Fast but insecure.
- **HTTPS** = HTTP + TLS encryption. Slightly slower but **mandatory** for production.

### 🔹 DNS Resolution
```
User types "google.com"
     ↓
Browser checks local cache
     ↓
Asks Recursive Resolver → Root DNS → TLD DNS → Authoritative DNS
     ↓
Gets IP address → connects to server
```

### 🔹 TCP vs UDP

| | TCP | UDP |
|-|-----|-----|
| **Reliability** | ✅ Guaranteed delivery | ❌ Best-effort |
| **Speed** | Slower (handshake overhead) | Faster |
| **Use case** | HTTP, file transfer, DB | Video streaming, gaming, DNS |

### 🔹 REST APIs vs gRPC

| | REST | gRPC |
|-|------|------|
| **Format** | JSON (text) | Protobuf (binary) |
| **Speed** | Slower | Much faster |
| **Best for** | Public APIs, browsers | Internal microservices |

### 🔹 CDN (Content Delivery Network)
- A network of servers placed **globally** that cache static content (images, JS, CSS, videos)
- User gets served from the **nearest CDN node** → lower latency
- Examples: Cloudflare, AWS CloudFront, Akamai

---

## 3️⃣ Databases & Storage Systems

> The most critical choice in any system design. Wrong DB = future pain.

### 🔹 SQL — When to Use

**Core concepts to know:**
- **Normalization** — eliminate redundancy, organize data into tables
- **Joins** — combine data from multiple tables
- **Indexes** — B-Tree (range queries), Hash (exact lookups) — speeds up reads massively
- **Transactions & Isolation Levels** — prevent dirty reads, phantom reads
- **Query Optimization** — EXPLAIN plans, avoid N+1 queries

### 🔹 NoSQL — Types & When to Use

| Type | DB | Use Case |
|------|----|----------|
| **Document** | MongoDB | Flexible schemas, user profiles |
| **Key-Value** | Redis | Caching, sessions, rate limiting |
| **Column Store** | Cassandra | Time-series, write-heavy analytics |
| **Graph** | Neo4j | Social networks, fraud detection |

### 🔹 Advanced Database Concepts

#### Sharding — Splitting data horizontally across multiple DBs
```
Shard 1: UserID 1–1M       Range-based sharding
Shard 2: UserID 1M–2M
Shard 3: UserID 2M–3M

OR

Shard = hash(userID) % N   Hash-based sharding (more even)
```

| Strategy | Pros | Cons |
|----------|------|------|
| Range-based | Simple | Hotspots possible |
| Hash-based | Even distribution | Range queries hard |
| Directory-based | Flexible | Directory becomes bottleneck |

#### Replication — Copies of data across machines
- **Master-Slave**: One master writes, many slaves read → great for read-heavy systems
- **Multi-Master**: Multiple masters write → complex conflict resolution

#### Consistency Models
- **Strong Consistency** — reads always see the latest write (slower)
- **Eventual Consistency** — reads may be stale temporarily, but will sync (faster, available)

> 🧠 **Practice**: Design a database schema for Instagram — Users, Posts, Follows, Likes, Comments tables.

---

## 4️⃣ Caching

> Caching is the single most impactful performance optimization in system design.

### 🔹 Why Cache?
Database reads are slow (disk I/O). A cache stores frequently accessed data **in memory** so you don't hit the DB every time. Result: **100x speed boost**.

### 🔹 Types of Caching

| Type | Example | What it caches |
|------|---------|----------------|
| **Application Cache** | In-process (e.g., HashMap) | Computed results |
| **Distributed Cache** | Redis, Memcached | Session data, API results |
| **CDN Cache** | Cloudflare | Static assets (images, JS) |
| **DB Query Cache** | MySQL Query Cache | Frequent DB queries |

### 🔹 Cache Write Strategies

| Strategy | How it works | Best for |
|----------|--------------|----------|
| **Write-Through** | Write to cache AND DB simultaneously | Read-heavy, data must be consistent |
| **Write-Around** | Write to DB only, bypass cache | Write-heavy, not read again soon |
| **Write-Back** | Write to cache first, DB later (async) | Write-heavy, tolerate some data loss risk |

### 🔹 Cache Eviction Policies

| Policy | Evicts | Use case |
|--------|--------|----------|
| **LRU** (Least Recently Used) | Oldest unused item | General purpose — most common |
| **LFU** (Least Frequently Used) | Item used least often | When access frequency matters |
| **FIFO** (First In, First Out) | Oldest inserted item | Simple queue-based workloads |

```
LRU Example (capacity = 3):
Access: A → B → C → A → D
Cache: [A, B, C] → A moves to front → [A, B, C] → D evicts B (LRU)
Result: [D, A, C]
```

> 🧠 **Practice**: Implement an LRU Cache in code using a HashMap + Doubly Linked List.

---

## 5️⃣ Message Queues & Event Systems

> Decouple your services. Never let a slow consumer crash a fast producer.

### 🔹 The Core Problem
Without queues: `Producer → directly calls → Consumer`
If consumer is slow or down → producer is blocked or data is lost.

With queues: `Producer → Queue → Consumer`
The queue **buffers** the messages. Producer and consumer are fully decoupled.

### 🔹 Tools

| Tool | Best for |
|------|----------|
| **Kafka** | High-throughput event streaming, log aggregation |
| **RabbitMQ** | Task queues, reliable message delivery |
| **SQS** (AWS) | Simple, managed, cloud-native queuing |
| **Google Pub/Sub** | GCP-native pub/sub messaging |

### 🔹 Core Concepts

- **Producer / Consumer model** — producer writes, consumer reads
- **Partitioning** — messages split across partitions for parallelism
- **Consumer Groups** — multiple consumers share work from one topic
- **Message Offset** — position of a message in a partition (Kafka)
- **At-least-once delivery** — message delivered 1+ times (duplicates possible)
- **Exactly-once delivery** — message delivered exactly 1 time (harder, slower)
- **Dead Letter Queue (DLQ)** — failed messages go here for inspection/retry

```
Producer ──▶ [Kafka Topic]
               ├── Partition 0 ──▶ Consumer Group A (Consumer 1)
               ├── Partition 1 ──▶ Consumer Group A (Consumer 2)
               └── Partition 2 ──▶ Consumer Group A (Consumer 3)
```

> 🧠 **Practice**: Design an email notification system — order placed → Kafka → email service. (See detailed design in [HLD_LEC_7_Notification_System.md](file:///c:/Users/saura/system-design/HLD_LEC_7_Notification_System.md)).

---

## 6️⃣ Load Balancing

> Traffic coming to your system needs to be distributed. One server is never enough.

### 🔹 What is a Load Balancer?
A load balancer sits **in front of your servers** and distributes incoming requests so no single server gets overwhelmed.

```
                    ┌──▶ Server 1
Client ──▶ [LB] ───┼──▶ Server 2
                    └──▶ Server 3
```

### 🔹 Load Balancing Algorithms

| Algorithm | How it works | Best for |
|-----------|--------------|----------|
| **Round Robin** | Send to each server in order 1→2→3→1→2→3 | Equal server capacity |
| **Least Connections** | Send to server with fewest active connections | Long-lived connections |
| **Weighted** | Servers get traffic proportional to their weight | Mixed server capacity |
| **IP Hash** | Hash client IP → always same server | Session affinity |

### 🔹 Types of Load Balancers

| Type | Layer | What it sees |
|------|-------|--------------|
| **L4 LB** | Transport | IP + Port only (fast, simple) |
| **L7 LB** | Application | HTTP headers, cookies, URLs (smart routing) |
| **Reverse Proxy** (NGINX) | Application | Acts as LB + caching + SSL termination |

### 🔹 Key Concepts

- **Health Checks** — LB pings servers; removes unhealthy ones automatically
- **Failover** — traffic auto-reroutes if a server dies
- **Sticky Sessions** — user always goes to the same server (needed for stateful apps)

> 🧠 **Practice**: Use NGINX to load-balance 2 Node.js servers locally.

---

## 7️⃣ Rate Limiting & Throttling ⭐ Deep Dive

> This is the **most asked topic** in system design interviews. Master every algorithm.

---

### 🔹 What is Rate Limiting?

Rate Limiting controls **how many requests a client can send to a server within a given timeframe**.

**Real-world problem without it:**
```
Client 1 ──▶ 10,000 req/sec ──▶  💥 Server crashes
Client 2 ──▶ 5,000 req/sec  ──▶
```

**Problems caused:**
1. **DDoS / DoS attacks** — attacker floods server with requests
2. **Server cost explosion** — every request costs money at scale
3. **Fair access denied** — one client monopolizes all resources

---

### 🔹 Where to Apply Rate Limiting?

```
Client → [Frontend] → [Middleware / API Gateway ✅] → [Backend]
```

| Layer | Should you use? | Why |
|-------|-----------------|-----|
| **Client Side** | ❌ Never rely on this | Easily bypassed — client controls it |
| **Server Side** | ✅ Yes | Reliable, enforced server-end |
| **Middleware / API Gateway** | ✅ Best | Centralized, scalable, decoupled |

**Real implementations:**
- Amazon AWS → **API Gateway** (built-in rate limiting)
- Shopify → **Leaky Bucket** at API level
- Custom → Build with **Redis + Fixed Window**

---

### 🔹 Rate Limiting — On What Basis?

| Basis | Use Case |
|-------|----------|
| **IP Address** | Limit per unique IP — stops anonymous attacks |
| **User ID** | Limit per logged-in user — fair per-user quota |
| **API Key** | Limit per application — SaaS multi-tenant |
| **Endpoint** | Different limits per route (100 APIs → per-API limits) |

---

### 🔹 HTTP Codes & Headers

```http
HTTP 429 Too Many Requests

Headers:
X-RateLimit-Limit     : 100       → max requests allowed
X-RateLimit-Remaining : 0         → requests left
X-RateLimit-Reset     : 1716400  → when counter resets (epoch)
Retry-After           : 30        → seconds to wait
```

---

### 🔹 What is Throttling? (vs Rate Limiting)

> People confuse these. Here's the **exact difference**:

| | Rate Limiting | Throttling |
|-|---------------|------------|
| **What it does** | Hard cap on number of requests per period | Controls the **rate/speed** of processing |
| **Reaction** | Reject after N requests | Slow down, queue, or delay requests |
| **Trigger** | Quota exceeded | System load is high |
| **Example** | "You get 1000 req/hour max" | "Slow requests down when CPU hits 80%" |

**In short:**
- Rate Limiting = **HOW MANY** (quota enforcement)
- Throttling = **HOW FAST** (flow control)

> Most production systems use **both together**: rate limit to cap quota, throttle to smooth spikes.

**Types of Throttling:**

| Type | Description |
|------|-------------|
| **API Throttling** | Limit req/sec per client to protect backend |
| **Network Throttling** | Limit bandwidth (upload/download) — ISPs, testing |
| **Resource Throttling** | CPU/memory limits in containers (Kubernetes) |

---

### 🔹 The 5 Rate Limiting Algorithms

---

#### 1️⃣ Token Bucket Algorithm

> Used by: **Amazon AWS**. Best for: allowing controlled bursts.

**Mental Model**: Imagine a bucket that gets tokens dropped in at a fixed rate. Each request costs one token. No token = no service.

```
                  [Token Builder]
                  adds 10 tokens/sec
                        │
                        ▼
                   ┌─────────┐
  Request ──▶ ─── │  Bucket  │ ──▶ Server ✅
  (takes 1 token) │ (max 20) │
                   └────┬────┘
                        │ Empty? → Drop ❌
                        ▼
                     req4 DROPPED
```

**Rules:**
1. Token builder pushes tokens into bucket at a **fixed rate** (e.g., 10/sec)
2. Bucket has a **max capacity** (e.g., 20 tokens)
3. New tokens when bucket is full → **overflow and discard**
4. Each incoming request **consumes 1 token**
5. No tokens available → **request is dropped**

**Config example:**
```
Bucket Size  = 20 tokens   (burst allowance)
Inflow Rate  = 10 tok/sec  (sustained rate)
```

| ✅ Pros | ❌ Cons |
|---------|---------|
| Simple to implement | Hard to tune bucket size + inflow rate |
| Allows short **burst traffic** (up to bucket size) | Tuning wrong → either too strict or too lenient |
| Memory efficient | |

---

#### 2️⃣ Leaky Bucket Algorithm

> Used by: **Shopify**. Best for: enforcing a **constant, smooth output rate**.

**Mental Model**: Water pours into a bucket with a hole at the bottom. The hole leaks at a constant rate. Overfill → overflow and spill.

```
Requests ──▶ ┌──────────┐
  (any rate)  │  Queue   │ ──▶ Server (2 req/sec constant)
              │ (cap: 3) │
              └────┬─────┘
    Overflow ──▶ DROP ❌
```

**Rules:**
1. Requests enter a **fixed-size queue** (bucket)
2. Requests are **processed at a constant outflow rate**
3. If the bucket is full → incoming requests are **dropped**
4. Regardless of how fast requests pour in, output is **always steady**

**Config example:**
```
Bucket Capacity = 3 requests
Outflow Rate    = 2 req/sec
```

| ✅ Pros | ❌ Cons |
|---------|---------|
| Simple to implement | During DDoS, queue fills with **attacker requests** |
| Server **never gets overwhelmed** | Legitimate requests **starve** (can't get into the queue) |
| Smooth, predictable output rate | Burst traffic is **not handled** gracefully |

> 🔑 **Key insight**: Leaky Bucket is great for protecting the server but terrible during actual attacks — the attacker's requests fill the queue, killing real user traffic.

---

#### 3️⃣ Fixed Window Counter

> Simple, fast. Best for: basic rate limiting with simple implementation.

**Mental Model**: Divide time into fixed buckets (windows). Count requests in each window. If count exceeds limit → reject.

```
│←── Window 1 ──→│←── Window 2 ──→│←── Window 3 ──→│
│   req 1,2,3    │   req 1,2,3    │   req 1,2,3    │
│   (max = 3)    │   (max = 3)    │   (max = 3)    │
    00:00–01:00      01:00–02:00      02:00–03:00
```

**Rules:**
1. Time is divided into **fixed intervals** (e.g., 1 second, 1 minute)
2. A counter tracks requests per window
3. If counter exceeds limit → **reject with 429**
4. Counter **resets at window boundary**

**The Critical Bug — Edge Case:**
```
Limit = 3 req/minute
Window resets at :00 and :60

User sends:
  3 requests at :59  (allowed — end of window 1)
  3 requests at :01  (allowed — start of window 2)

Result: 6 requests in 2 seconds → server gets 2x the allowed load!
```

```
         Window 1              Window 2
│────────────────:59│:01────────────────│
               ↑↑↑   ↑↑↑
         3 req here   3 req here = 6 in 2 seconds 💥
```

| ✅ Pros | ❌ Cons |
|---------|---------|
| Easiest to implement | **Burst at window boundary** — classic edge case bug |
| Very memory-efficient (1 counter) | Not accurate for strict rate limiting |
| Fastest | Vulnerable to timing attacks |

---

#### 4️⃣ Sliding Window Log Algorithm

> 🏆 **Most Accurate** algorithm. Best for: strict, precise rate limiting.

**Mental Model**: Instead of resetting a counter, you keep a **log of every request's timestamp**. When a new request comes in, you look back exactly 1 window and count.

```
Log (timestamps):  [00:01, 00:30, 00:59, 01:02, ...]
                                           ↑
New request at 01:02 arrives.

Step 1: Remove all entries older than (01:02 - 1 minute) = 00:02
        → Remove 00:01 (it's now expired)
        Log is now: [00:30, 00:59, 01:02]

Step 2: Count entries in log = 3

Step 3: Limit is 3 → count (3) ≤ limit (3) → ✅ Allow
        Add 01:02 to log
```

**Algorithm step-by-step:**
1. Store each request as a **timestamp in a log** (sorted set)
2. When new request arrives → **delete all timestamps** older than `now - window_size`
3. Count remaining entries in log
4. If count < limit → **allow** + add timestamp to log
5. If count ≥ limit → **reject (429)**

> Uses **UNIX/Linux EPOCH timestamps** (millisecond precision) for accuracy.

| ✅ Pros | ❌ Cons |
|---------|---------|
| **Most accurate** — no edge case bursts | **Memory intensive** — stores every request timestamp |
| Works correctly at any point in time | **Slower** — must clean + count log on every request |
| No boundary spike issues | For high traffic systems, log size grows fast |

---

#### 5️⃣ Sliding Window Counter Algorithm (Hybrid)

> Best of both worlds. Used in: **most production systems**. Best for: balancing accuracy and memory.

**Mental Model**: Instead of storing every timestamp (log), keep just **two window counters** and estimate the sliding window count mathematically using a weighted formula.

```
│←────── Previous Window ─────→│←── Current Window ──→│
│         5 requests            │     4 requests        │
│                               │                       │
└──────────────────[now - 1min]──────[now]──────────────┘
                        ↑
              Rolling window boundary
              Currently 70% into previous window
```

**The Formula:**
```
Estimated count = (Requests in current window)
                + (Requests in previous window × overlap_percentage)
```

**Example** — Limit = 7 req/minute:
```
Current window:  4 requests
Previous window: 5 requests
Rolling window is 30% into current window (70% of previous still applies)

Estimated = 4 + (5 × 0.70)
           = 4 + 3.5
           = 7.5

7.5 > 7 → ❌ REJECT (429)
```

```
│← prev (70%) →│← curr (30%) →│
  5 * 0.7 = 3.5   + 4 = 7.5   > 7 → DROP
```

| ✅ Pros | ❌ Cons |
|---------|---------|
| Much more memory efficient than log | Approximate — not 100% precise |
| Better accuracy than fixed window | Slightly complex formula |
| Handles bursts well | Small error margin (~0.003% — negligible in practice) |

---

### 🔹 Build Your Own Rate Limiter (System Design)

**Algorithm**: Fixed Window Counter (simple, fast, easy to explain)

**Full Architecture:**

```
                        DB Rules (config)
                              │
Client ──▶ [API Gateway] ──▶ [Rate Limiter Middleware]
                                     │
                             ┌───────┴────────┐
                             │   Redis Cache   │  ← INCR command
                             │  (in-memory)   │
                             └───────┬────────┘
                                     │
                    ┌────────────────┴──────────────────┐
                    │                                   │
              Counter < limit                    Counter ≥ limit
                    │                                   │
             ✅ Forward to Server             ❌ Return 429
                    │
            [Message Queue / Kafka]
                    │
            [Backend Workers]
```

**Why Redis, not a database?**

| Storage | Why bad/good for rate limiting |
|---------|-------------------------------|
| **SQL/NoSQL DB** | ❌ Too slow — disk I/O, high latency under load |
| **Redis (in-memory)** | ✅ Microsecond latency, atomic operations, perfect |

**Redis command:**
```redis
INCR user:123:counter       → atomically increment counter by 1
EXPIRE user:123:counter 60  → auto-reset after 60 seconds
```

**Flow:**
1. Request arrives → Rate Limiter hits Redis
2. `INCR` the counter key for `(userId + window)`
3. If new counter ≤ limit → **allow**, forward request
4. If new counter > limit → **reject 429**, return error
5. Redis key auto-expires after the window duration

---

### 🔹 Algorithm Comparison — Full Summary

| Algorithm | Accuracy | Memory | Speed | Burst OK? | Used By |
|-----------|----------|--------|-------|-----------|---------|
| **Token Bucket** | Medium | Low | Fast | ✅ Yes | Amazon |
| **Leaky Bucket** | Medium | Low | Fast | ✅ Smoothed | Shopify |
| **Fixed Window** | Low | Lowest | Fastest | ❌ Edge burst | Simple systems |
| **Sliding Window Log** | **Highest** | High | Slow | ✅ Yes | Strict APIs |
| **Sliding Window Counter** | High | Medium | Medium | ✅ Yes | Most prod systems |

**How to choose:**

```
Need simplicity?              → Fixed Window Counter
Need burst support?           → Token Bucket
Need smooth output?           → Leaky Bucket
Need maximum accuracy?        → Sliding Window Log
Need accuracy + efficiency?   → Sliding Window Counter ✅ (recommended)
```

---

### 🔹 Throttling — Where It Fits in System Design

Throttling protects **every layer** of your stack:

| Layer | How throttling helps |
|-------|---------------------|
| **API Gateway** | Limits req/sec per client → protects all downstream services |
| **Microservices** | Each service throttles independently → prevents cascading failures |
| **Network / CDN** | Bandwidth throttling → prevents one client hogging the pipe |
| **Kubernetes / Containers** | CPU/memory throttling → prevents noisy-neighbor problems |
| **Database** | Connection pool throttling → prevents DB from getting overwhelmed |
| **Security** | Throttle per IP → mitigate DDoS/brute force attacks |

> 💡 **Design principle**: Throttling is not an afterthought. Embed it from day 1. Retrofitting it later into a high-traffic system is extremely painful.

---

## 8️⃣ Distributed System Concepts

> The hard stuff. Interviewers love these.

### 🔹 Leader Election
When you have multiple nodes, only **one leader** should perform certain operations (writes, coordination). Algorithms:
- **Raft** — simpler, widely used (used by etcd, CockroachDB)
- **Paxos** — more complex, foundational theory

### 🔹 Consensus Algorithms
Multiple nodes must **agree on a value** even if some nodes fail. Used in: distributed databases, distributed locks.

### 🔹 Heartbeats & Failure Detection
Nodes send periodic "I'm alive" signals. If no heartbeat within timeout → node is assumed dead. System reroutes.

### 🔹 Gossip Protocol
Nodes randomly share state with neighbors. Information **spreads like gossip** across the cluster without a central coordinator. Used by: Cassandra, DynamoDB.

### 🔹 Vector Clocks
Track **causality** of events in a distributed system. Answers: "Which event happened before which?" without a global clock.

### 🔹 Distributed Locks
Ensure **only one process** does something at a time across multiple machines:
- **Redis** → `SET key value NX PX 30000` (lock with expiry)
- **Zookeeper** → Ephemeral nodes for distributed locking

---

## 9️⃣ HLD Architecture Patterns

> These are the patterns that separate junior from senior engineers.

### 🔹 Microservices Architecture
Break app into **small, independently deployable services** that communicate via APIs or message queues.

### 🔹 Event-Driven Architecture
Services communicate by **publishing and consuming events** (not direct calls). Enables loose coupling.

```
Order Service ──▶ [Kafka: "order.placed"] ──▶ Inventory Service
                                          └──▶ Email Service
                                          └──▶ Analytics Service
```

### 🔹 API Gateway Pattern
Single entry point for all clients. Handles: routing, auth, rate limiting, SSL termination, request transformation.

### 🔹 CQRS (Command Query Responsibility Segregation)
Separate **write operations** (Commands) from **read operations** (Queries) using different models/databases.

```
Client ──▶ Command API ──▶ Write DB (normalized SQL)
       └──▶ Query API  ──▶ Read DB (denormalized, optimized)
```

### 🔹 Event Sourcing
Instead of storing current state, store the **full history of events**. Current state = replay all events.

### 🔹 Saga Pattern
Manage distributed transactions across microservices:
- **Orchestration** — central coordinator tells each service what to do
- **Choreography** — each service reacts to events and triggers the next step

### 🔹 Sidecar Pattern
Deploy a helper container **alongside** your main container (same pod in Kubernetes). Handles: logging, monitoring, service mesh (Istio).

### 🔹 Strangler Fig Pattern
Gradually **replace a monolith** with microservices by routing traffic incrementally. Old system "strangles" as new services take over.

---

## 🔟 Low-Level Design (LLD)

> LLD = Class design, OOP, design patterns. Interviews test this separately.

### 🔹 SOLID Principles

| Principle | What it means |
|-----------|---------------|
| **S**ingle Responsibility | One class, one job |
| **O**pen/Closed | Open for extension, closed for modification |
| **L**iskov Substitution | Subclass can replace parent class |
| **I**nterface Segregation | Small, focused interfaces |
| **D**ependency Inversion | Depend on abstractions, not concretions |

### 🔹 Design Patterns to Know

| Pattern | What it does | Example |
|---------|--------------|---------|
| **Factory** | Creates objects without specifying class | `DBConnectionFactory` |
| **Singleton** | Only one instance of a class | Logger, Config |
| **Observer** | Objects subscribe to events | Event listeners |
| **Strategy** | Swap algorithms at runtime | Payment methods, sorting |

> 🧠 **Practice**: Design a Parking Lot, Hotel Booking system, or LRU Cache class.

---

## 1️⃣1️⃣ Scalability & Deployment

### 🔹 Horizontal Scaling + Auto-Scaling
Add servers automatically based on load. AWS Auto Scaling Groups, Kubernetes HPA.

### 🔹 Canary Deployment
Release new version to **1–5% of users** first. Monitor. If good → roll out to everyone.

### 🔹 Blue-Green Deployment
Two identical environments (Blue = live, Green = new). Switch traffic instantly with zero downtime.

### 🔹 Containerization
- **Docker** — package app + dependencies into a portable container
- **Kubernetes** — orchestrate containers at scale (Pods, Deployments, Services, Ingress)

---

## 1️⃣1️⃣ Monitoring & Observability

> If you can't measure it, you can't improve it.

### 🔹 The 3 Pillars

| Pillar | Tool | What it tracks |
|--------|------|----------------|
| **Metrics** | Prometheus + Grafana | CPU, memory, req/sec, error rate |
| **Logs** | ELK Stack (Elasticsearch, Logstash, Kibana) | Application events, errors |
| **Traces** | Jaeger, OpenTelemetry | Request flow across microservices |

### 🔹 Key Metrics to Monitor
- **Latency** (p50, p95, p99)
- **Error Rate** (4xx, 5xx)
- **Throughput** (RPS)
- **Saturation** (CPU %, memory %)

---

## 1️⃣2️⃣ System Design Interview Questions

### Beginner
| # | Problem | Key components |
|---|---------|----------------|
| 1 | **URL Shortener** | Hash function, KV store, redirect |
| 2 | **Rate Limiter** | Token bucket / Redis counter |
| 3 | [**Notification System**](file:///c:/Users/saura/system-design/HLD_LEC_7_Notification_System.md) | Kafka, push/email/SMS service |
| 4 | **Chat System** | WebSockets, message queue |
| 5 | **News Feed** | Fan-out, caching, timeline |

### Intermediate
| # | Problem | Key challenge |
|---|---------|---------------|
| 6 | **Instagram** | Image storage, CDN, feed generation |
| 7 | **WhatsApp** | Real-time messaging, end-to-end encryption |
| 8 | **YouTube** | Video encoding, CDN, recommendation |
| 9 | **Twitter Timeline** | Fan-out on write vs read |
| 10 | **E-commerce** | Inventory, payments, order management |
| 11 | **Real-time Location** | Geospatial index, WebSocket |
| 12 | **Search Autocomplete** | Trie, Redis sorted sets |

### Advanced
| # | Problem | Key challenge |
|---|---------|---------------|
| 13 | **Uber** | Geospatial matching, surge pricing |
| 14 | **Netflix** | Adaptive streaming, global CDN |
| 15 | **Zoom** | WebRTC, low-latency video |
| 16 | **TikTok** | Recommendation ML, video pipeline |
| 17 | **Global CDN** | PoP placement, cache invalidation |
| 18 | **Payment Gateway** | ACID transactions, idempotency |
| 19 | **Distributed Cache** | Consistent hashing, eviction |
| 20 | **Multiplayer Game** | Low-latency sync, game state |

### 🧠 How to answer any HLD question (Framework):

```
1. Requirements        → Clarify functional + non-functional
2. Capacity Estimation → Users, RPS, storage, bandwidth
3. High-Level Design   → Draw the big boxes + arrows
4. Deep Dive           → Zoom into 1–2 critical components
5. Database Schema     → Tables, indexes, sharding strategy
6. Scaling Strategy    → How does this handle 10x traffic?
7. Bottlenecks         → What breaks first? How to fix?
```

---

## 🗂 1-Month Study Plan

| Week | Focus | Topics |
|------|-------|--------|
| **Week 1** | Foundations | Basics + Networking + DB Fundamentals |
| **Week 2** | Core Components | Caching + Queues + Load Balancing |
| **Week 3** | Advanced | Distributed Systems + Architecture Patterns |
| **Week 4** | Practice | HLD Practice + LLD Practice + Mock Interviews |

---

> 📌 **Remember**: System design has no single right answer. The best answer shows your **thought process**, **trade-offs awareness**, and **ability to scale**. Always justify your choices.
