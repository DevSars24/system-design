# 🚦 HLD Lecture 5 — Rate Limiting & Caching Strategies

---

## 📌 What is Rate Limiting?

Rate Limiting controls **how many requests a client can send to a server within a given timeframe**.

### Problems Without Rate Limiting:
1. **Client can send too many requests** within a timeframe → leads to **DDoS / DoS attacks**
2. **Server cost becomes expensive** — each request is costly at scale

---

## 🏗️ Where to Apply Rate Limiting?

```mermaid
flowchart LR
    C[Client] --> FE[Frontend]
    FE --> GW["✅ API Gateway / Middleware\n(Best Place — centralized, scalable)"]
    GW --> BE[Backend]

    C2[Client] -.->|❌ Client-side only| FE
    note["Client-side: Easily bypassed\nServer-side: Reliable\nAPI Gateway: Best practice"]
```

| Layer | Notes |
|-------|-------|
| **Client Side** | ❌ Worst — can be bypassed easily |
| **Server Side** | ✅ More reliable |
| **Middleware / API Gateway** | ✅ Best practice for large-scale systems |

---

## 🛠️ Application Use Cases

1. **Server-Side Language** (Java, C++, JavaScript, Python) — implement rate limiting logic directly
2. **3rd Party Rate Limiter Providers** — e.g., Amazon AWS API Gateway

---

## 🔑 Rate Limiting — On What Basis?

| Basis | Description |
|-------|-------------|
| **IP Address** | Limit per unique IP |
| **User ID** | Limit per authenticated user |
| **Other Keys** | Primary key, candidate key, API key, etc. |

---

## 📡 HTTP Response Codes for Rate Limiting

| Code | Meaning |
|------|---------|
| `200` | OK |
| `429` | **Too Many Requests** |

### Relevant HTTP Headers:
```
X-api-limit : 2          → max 2 requests allowed
X-duration  : seconds    → per this time window
```

---

## 🧮 Algorithms of Rate Limiting

1. **Token Bucket**
2. **Leaky Bucket**
3. **Fixed Window Counter**
4. **Sliding Window Log**
5. **Sliding Window Counter**

---

### 1️⃣ Token Bucket Algorithm

```mermaid
flowchart LR
    TB["Token Builder\n(adds 10 tokens/sec)"] -->|fills| B["Bucket\n(max 20 tokens)"]
    R[Incoming Request] -->|takes 1 token| B
    B -->|token available| S[Server ✅]
    B -->|bucket empty| D[Drop Request ❌]
    B -->|bucket full, new tokens| OV[Overflow — discard tokens]
```

**Configuration:**
- `Bucket Size` : 20 tokens
- `Inflow Rate` : 10 tokens/sec

**How it works:**
1. Token builder pushes tokens into the bucket at fixed intervals
2. If bucket is **full**, new tokens are **discarded** (overflow)
3. When a request comes → it takes a token from the bucket → goes to server
4. If bucket is **empty** → request is **dropped**

| | |
|-|-|
| **Used by** | Amazon |
| ✅ **Pros** | Simple to implement; can handle **burst traffic** for short durations |
| ❌ **Cons** | Hard to decide optimal bucket capacity and inflow rate |

---

### 2️⃣ Leaky Bucket Algorithm *(Global Rate Limiting)*

```mermaid
flowchart LR
    R1[Request surge] --> Q["Queue / Bucket\n(capacity: 3)"]
    R2[More requests] --> Q
    Q -->|constant outflow: 2 req/sec| S[Server]
    Q -->|bucket full → overflow| D[Drop ❌]
```

**Configuration:**
- `Bucket Capacity` : 3
- `Outflow Rate` : 2 req/sec

**How it works:**
- Requests fill a queue (bucket); they are processed at a **steady outflow rate**
- Excess requests that overflow the bucket are **dropped**

| | |
|-|-|
| **Used by** | Shopify |
| ✅ **Pros** | Simple; protects server even during burst traffic |
| ❌ **Cons** | During DDoS, the queue fills with attack requests → **valid requests starve** |

---

### 3️⃣ Fixed Window Counter

```mermaid
gantt
    title Fixed Window Counter (limit = 3 req/window)
    dateFormat X
    axisFormat s%s

    section Window 1 (00-60s)
    req1 :0, 1
    req2 :20, 21
    req3 :40, 41

    section Window 2 (60-120s)
    req1 :60, 61
    req2 :80, 81
    req3 :100, 101
```

**Fixed Counter = 3**

**⚠️ Edge Case — Burst at Window Boundary:**
```
Limit = 3 req/minute
User sends 3 req at :59 (end of Window 1) ✅ allowed
User sends 3 req at :01 (start of Window 2) ✅ allowed
→ 6 requests in 2 seconds — server takes 2x the load! 💥
```

| | |
|-|-|
| ✅ **Pros** | Simple to implement |
| ❌ **Cons** | **Burst at window edges** — e.g., 3 req at end + 3 req at start = 6 in a short span |

---

### 4️⃣ Sliding Window Log Algorithm

> 🏆 **Best Algorithm for Rate Limiting** — Strict but slow & memory-consuming

```mermaid
sequenceDiagram
    participant R as Request (01:02)
    participant L as Log
    participant S as Server

    R->>L: New request arrives at 01:02
    L->>L: Remove all entries older than (01:02 - 1min) = 00:02
    Note over L: Removed: [00:01]. Remaining: [00:30, 00:59]
    L->>L: Add new entry: 01:02
    Note over L: Log = [00:30, 00:59, 01:02] → count = 3
    L->>S: count(3) ≤ limit(3) → ✅ Allow
```

> Uses **LINUX EPOCH timestamps** for precision

| | |
|-|-|
| ✅ **Pros** | Most accurate; no edge-case bursts |
| ❌ **Cons** | Slow & memory-intensive (stores every request log) |

---

### 5️⃣ Sliding Window Counter Algorithm *(Hybrid Approach)*

Combines Fixed Window + Sliding Window Log for a **balanced** approach.

**Formula:**
```
Estimated Requests = (Req in current window) + (Req in previous window × overlap%)
```

**Example** (7 req/minute limit, 70% overlap):
```
= 4 (current) + 5 (previous) × 0.7
= 4 + 3.5
= 7.5  →  over limit → drop ❌
```

```mermaid
xychart-beta
    title "Sliding Window Counter — Rolling Overlap"
    x-axis ["Prev Window (70% applies)", "Current Window (100%)"]
    y-axis "Requests" 0 --> 8
    bar [3.5, 4]
    line [7, 7]
```

---

## 🏗️ Building Your Own Rate Limiter

### Architecture

```mermaid
flowchart TD
    C[Client] --> RL["Rate Limiter\n(Middleware)"]
    RL --> R[(Redis Cache\nIn-Memory, Fast)]
    R --> DBR[(DB Rules\nconfig)]
    R -->|counter < limit| S[Server ✅]
    R -->|counter ≥ limit| E["429 Too Many Requests ❌"]
    S --> MQ[Message Queue / Kafka]
    MQ --> W[Backend Workers]
```

### Why Redis over DB?
| Storage | Speed |
|---------|-------|
| **DB (SQL/NoSQL)** | ❌ Slow for rate limiting |
| **Redis (In-Memory Cache)** | ✅ Fast — ideal for rate limiting |

### Redis Command Used:
```redis
INCR   →  Atomically increment counter in Redis
EXPIRE →  Auto-reset key after window duration
```

### Flow:
1. Request comes in → **Rate Limiter** checks Redis
2. Redis counter < limit → **forward to server**
3. Redis counter ≥ limit → **return `429 Too Many Requests`**
4. Workers and Kafka handle async processing via **Messaging Queue**

---

## 📊 Algorithm Comparison Summary

| Algorithm | Accuracy | Memory | Speed | Burst Handling |
|-----------|----------|--------|-------|----------------|
| Token Bucket | Medium | Low | Fast | ✅ Yes |
| Leaky Bucket | Medium | Low | Fast | ✅ Smoothed |
| Fixed Window | Low | Very Low | Fastest | ❌ Edge bursts |
| Sliding Window Log | **High** | High | Slow | ✅ Yes |
| Sliding Window Counter | High | Medium | Medium | ✅ Yes |

---

*Source: HLD Lecture 5 — Rate Limiting & Caching Strategies*
