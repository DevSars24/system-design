# HLD Master Notes — Lecture 3
### High Level Design: CAP Theorem · Back-of-Envelope · Monolith vs Microservice

---

## Table of Contents
1. [CAP Theorem](#1-cap-theorem)
2. [Availability Numbers](#2-availability-numbers)
3. [Back of the Envelope — Instagram](#3-back-of-the-envelope--instagram)
4. [Monolith vs Microservice](#4-monolith-vs-microservice)
5. [Decomposition Phase](#5-decomposition-phase)
6. [Strangler Pattern](#6-strangler-pattern)
7. [DB Phase](#7-db-phase)
8. [Communication: Choreography vs Orchestration](#8-communication-choreography-vs-orchestration)
9. [CQRS + Master-Slave Model](#9-cqrs--master-slave-model)

---

## 1. CAP Theorem

> **"In any distributed system, you can only guarantee 2 out of 3 properties."**

| Property | Full Form | Meaning |
|---|---|---|
| **C** | Consistency | Every client gets the **same data** at the same time, no matter which node (DB) responds |
| **A** | Availability | Every request **always gets a response** (may not be latest data) |
| **P** | Partition Tolerance | System keeps running even if **network between nodes breaks** |

### The Rule
```
Partition Tolerance (P) is MANDATORY — always choose it.
So real choice is always between:  C  vs  A
```

| Combination | What it means | Example |
|---|---|---|
| **CP** | Consistent + Partition Tolerant | DB1, DB2, DB3 — writes blocked if partition; reads always fresh. Used in banking. |
| **AP** | Available + Partition Tolerant | Always responds, but may serve stale data. Used in social media. |
| **CA** | Consistent + Available | Only works with no partitions — not practical for distributed systems |

### Consistency vs Availability (Plain English)
- **Consistency** → Response is **up-to-date, mandatory**. May block if nodes disagree.
- **Availability** → Response is **always given**, up-to-date is optional.

---

## 2. Availability Numbers

> How much downtime is acceptable per year?

| SLA | Downtime per year |
|---|---|
| 99% | ~3.65 days |
| 99.9% | ~8.77 hours |
| 99.99% | ~52 minutes |
| 99.999% | ~5 minutes ← Google, Amazon, Microsoft |

**More 9s = less downtime = higher engineering cost.**

---

## 3. Back of the Envelope — Instagram

> Rough estimation before designing a system. Tests if your architecture can handle scale.

### Power of 2 Reference
| Power | Value | Unit |
|---|---|---|
| 2^10 | Thousands | Kilobyte (KB) |
| 2^20 | Millions | Megabyte (MB) |
| 2^30 | Billions | Gigabyte (GB) |
| 2^40 | Trillions | Terabyte (TB) |
| 2^50 | Quadrillions | Petabyte (PB) |
| 2^60 | — | Exabyte (EB) |

### Assumptions
- Monthly Active Users: **~2 Billion**
- Daily Active Users (DAU): **60% of MAU = ~1.2 Billion**
- User checks feed **30 times/day** (average)
- User uploads **1 photo/reel per day** (average)
- Estimate for **5 years**

### QPS Calculations

**Feed View QPS:**
```
Daily feed requests = 1.2B × 30 = 36B requests/day
QPS = 36B / 86,400 sec ≈ 420,000 QPS
Peak QPS = 2 × 420K ≈ 840K/sec
```

**Upload QPS:**
```
Daily upload requests = 1.2B × 1 = 1.2B/day
QPS = 1.2B / 86,400 ≈ 14,000 QPS
Peak QPS = 2 × 14K ≈ 28K/sec
```

### Storage Calculations

**Assumptions:**
- 80% users upload **photos** → avg 1 MB each
- 20% users upload **videos** → avg 50 MB each

**Photos/day:**
```
1.2B × 80% = ~1B photos × 1 MB = 1PB/day
```

**Videos/day:**
```
1.2B × 20% = 0.25B × 50MB ≈ 12PB/day
```

**Total daily storage:** `1PB + 12PB = 13PB/day`

**For 5 years:**
```
13PB × 365 × 5 ≈ 24,000 PB ≈ 24 Exabytes
```

---

## 4. Monolith vs Microservice

### Monolith (e.g. Tomato app — early days)

```
[ Single Codebase ]
  ├── Orders
  ├── Payments
  ├── Restaurants
  └── Users
```

| Pros | Cons |
|---|---|
| Fast (no network calls) | Hard to **scale** individual parts |
| Simple to build | **Heavy CI/CD** — entire app redeploys for 1 change |
| Easy local dev | **Debugging & testing** complex |

---

### Microservice (e.g. Zomato)

```
USERS --http--> Restaurants --http--> Orders
                                         |
                                      Payments
```

Each service:
- Has its **own codebase**
- Runs **independently**
- Communicates via **HTTP / Events**

| Pros | Cons |
|---|---|
| Independent deploy & scale | **Latency** (network hops) |
| Team autonomy | **Transaction management** hard |
| Tech freedom per service | **Multiple service JOIN** complex |

---

### 5 Phases of Microservice Migration

| Phase | What to decide |
|---|---|
| 1. **Decomposition** | How to break monolith into services |
| 2. **Database** | Shared DB or unique DB per service |
| 3. **Communication** | API calls or Event-driven |
| 4. **Deployment** | CI/CD pipeline per service |
| 5. **Observability** | Monitoring, logging, tracing |

---

## 5. Decomposition Phase

> How do you break a monolith into microservices?

### Pattern 1: Decomposition by Business Logic

Split by **what the code does** — Orders, Payments, Users, Restaurants each become a service.

**Disadvantage:** Every developer must understand the **entire business domain**.

---

### Pattern 2: Decomposition by Sub-domain ✅ (Recommended)

Split by **URL path / domain boundary:**

```
www.zomato.com/orders      → Orders Microservice
www.zomato.com/payments    → Payments Microservice
www.zomato.com/add-user    → Users Microservice
www.zomato.com/restaurants → Restaurants Microservice
```

**Advantage:** Each team owns exactly **one domain**. Orders team only knows Orders.

---

## 6. Strangler Pattern

> "Don't migrate all 100 APIs at once — do it gradually."

Named after the **strangler fig tree** 🌳 — a vine that slowly wraps around and replaces the host tree.

### How it works:

```
Step 1: Pick ONE route (e.g. /orders)
Step 2: Build a new microservice for it
Step 3: Redirect traffic to new service
Step 4: Test → then repeat for next route
```

### Real Example — Tomato App

```
Total: 100 APIs

Week 1:  Migrate /orders  → 10 APIs moved
         Client: 90 req → Monolith, 10 req → New service (10% done)

Week 3:  Migrate /payments
         Now: 40% on Microservices, 60% still on Monolith

Week 6:  100% migrated ✅
```

**Both systems run in parallel during migration.** Zero downtime approach.

---

## 7. DB Phase

> Each microservice can have: (1) Shared DB, or (2) Unique DB

### Option A — Shared DB ❌

All services read/write to **one central database.**

```
Orders svc ──┐
Payments svc─┼──► [Single DB]
Users svc ───┘
```

| Pros | Cons |
|---|---|
| Easy JOINs (SQL) | **Cannot scale** independently |
| Simple transactions (ACID) | One DB type for everyone (SQL or NoSQL — pick one) |
| Familiar | Schema change breaks all services |

---

### Option B — Unique DB per service ✅ (Recommended)

```
Orders svc   ──► Orders DB (PostgreSQL)
Payments svc ──► Payments DB (MySQL — ACID critical)
Users svc    ──► Users DB (MongoDB — flexible schema)
Restaurant   ──► Menu DB (Redis — fast reads/cache)
```

**This is called "Polyglot Persistence"** — use the **right DB for the right job.**

| Service | Best DB | Why |
|---|---|---|
| Orders | PostgreSQL | Structured, relational |
| Payments | MySQL | ACID transactions |
| Users | MongoDB | Flexible schema |
| Menu/Cache | Redis | Ultra-fast reads |

**Disadvantage:** JOINs across services are impossible → solved by **CQRS** (see section 9).

---

## 8. Communication: Choreography vs Orchestration

> When services need to coordinate (e.g. place order → charge payment → confirm restaurant), how do they talk?

---

### Choreography (Event Driven) 📡

**No central controller.** Each service publishes events and listens to others.

```
Order placed
    │
    ▼  [event: order_created]
Payment service
    │
    ▼  [event: payment_success]
Restaurant service
    │
    ▼  [event: confirmed]
Order complete ✅
```

**If Restaurant fails:**
```
Restaurant FAILS
    │
    └──► [rollback event] ──► Payment refunded
                           └──► Order cancelled
```

| Pros | Cons |
|---|---|
| Loose coupling | **Very complex to build** |
| No single point of failure | Hard to debug (no central view) |
| Scales well | Rollback logic spread across services |

---

### Orchestration (SAGA Pattern) 🎻

**One central Orchestrator** controls the flow via API calls.

```
           ┌──────────────────┐
           │   Orchestrator   │
           └────────┬─────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   1. Order     2. Payment  3. Restaurant
      svc          svc          svc
      [DB]         [DB]         [DB]
```

**If Restaurant fails:**
```
Restaurant FAILS → error to Orchestrator
Orchestrator → rollback Payment
Orchestrator → cancel Order
```

| Pros | Cons |
|---|---|
| **Central visibility** — easy to debug | **Single point of failure** |
| Explicit rollback logic | Orchestrator can become very large |
| Simple to reason about flow | Tight coupling to orchestrator |

---

## 9. CQRS + Master-Slave Model

> **Problem:** With separate DBs per service, SQL JOIN across services is impossible.
> **Solution:** CQRS — separate your read and write paths.

### CQRS = Command Query Responsibility Segregation

| Type | Operations | Where |
|---|---|---|
| **COMMAND** | INSERT, UPDATE, DELETE | Each service's own DB (Master) |
| **QUERY** | SELECT, JOIN | Shared View DB (Slave/Replica) |

### Master-Slave Architecture

```
Orders svc ──► Orders DB (Master) ──┐
Payments svc ─► Payments DB (Master)─┼──sync──► VIEW DB (Slave)
Restaurant svc ─► Restaurant DB ────┘              │
                                                    ▼
                                          SELECT * FROM orders
                                          JOIN payments ON ...
                                          JOIN restaurants ON ...
                                          ✅ JOIN works here!
```

### How it works:
- **Masters** handle all write operations (one per service)
- **Slave/View DB** is a **read-only replica** — synced from all masters
- All complex **READ queries and JOINs** go to the View DB
- This is the **Master-Slave Model** in action

| | Without CQRS | With CQRS |
|---|---|---|
| JOIN across services | ❌ Impossible | ✅ Easy via View DB |
| Write scalability | ❌ One DB bottleneck | ✅ Each service scales independently |
| Read scalability | ❌ Single DB load | ✅ Slave handles all reads |

---

## Quick Revision Cheatsheet

```
CAP Theorem      → Pick 2 of 3. P is mandatory. Choose C or A.
Availability     → More 9s = less downtime (99.999% = 5 min/year)
Back of Envelope → DAU × requests × size = QPS & storage estimate
Monolith         → Fast, simple, but hard to scale
Microservice     → Scalable, independent, but complex
Decomposition    → By Business Logic or Sub-domain (subdomain better)
Strangler        → Migrate gradually, route by route, not all at once
DB Phase         → Unique DB per service (Polyglot Persistence)
Choreography     → Event-driven, no boss, complex rollback
Orchestration    → Central boss (SAGA), explicit rollback, easier debug
CQRS             → COMMAND writes to master, QUERY reads from slave View DB
```

---

*Notes compiled from HLD Lecture 3 — Microservice Architecture Series*
