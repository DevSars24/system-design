# 🔗 System Design — URL Shortener (Bit.ly Clone)
> **HLD Lecture 6** · High-Level Design · Open-Ended Interview Question

---

## 📌 Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Clarifying Questions](#2-clarifying-questions)
3. [Back-of-the-Envelope Estimation](#3-back-of-the-envelope-estimation)
4. [API Design](#4-api-design)
5. [Redirection — 301 vs 302](#5-redirection--301-vs-302)
6. [Initial Flow Design](#6-initial-flow-design)
7. [Database Choice — NoSQL vs SQL](#7-database-choice--nosql-vs-sql)
8. [Short URL Generation Strategies](#8-short-url-generation-strategies)
9. [Distributed Architecture](#9-distributed-architecture)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Problem Statement

Design a **URL Shortener** service like **[Bit.ly](https://bit.ly)**.

```
Long URL  ──►  [Service]  ──►  Short URL
Short URL ──►  [Browser]  ──►  Redirected to Long URL
```

| Direction | Example |
|-----------|---------|
| Long → Short | `www.example.com/api/v1/userId=1&userName=Aditya` → `www.bit.ly/aQw12` |
| Short → Long | `www.bit.ly/aQw12` → browser redirects to original URL |

---

## 2. Clarifying Questions

These are the **first questions you must ask in an HLD interview**. They define the entire system scope.

### ❓ Q1 — Scale: How many URLs are we targeting?
> **Answer: 20 million URLs / day**

### ❓ Q2 — How short should the URL be?
> **Answer: As short as possible**

### ❓ Q3 — What characters can we use in the short URL?
> **Answer: `0–9`, `a–z`, `A–Z` → Total 62 characters**

---

## 3. Back-of-the-Envelope Estimation

This is a **critical skill** in system design interviews. Always calculate before designing.

### ⚡ QPS (Queries Per Second)

#### Write Operations (URL Generation)

```
20 million URLs/day
= 20 × 2²⁰ / 86,400 seconds
≈ 231 write operations/sec
```

#### Read Operations (URL Redirects)

> Assumption: **Write : Read = 1 : 10**  (reads always dominate in URL shorteners)

```
Read ops/sec = 231 × 10 = 2,310 ops/sec
```

### 💾 Storage Estimation (5 Years)

```
Records = 20 million/day × 365 days × 5 years
        = 20 × 2²⁰ × 1825
        ≈ 36.5 Billion records

Size per record = 100 bytes

Total Storage = 36.5 Billion × 100 bytes
              = 36.5 × 2³⁰ × 0.1 × 2¹⁰
              = 3.65 × 2⁴⁰
              ≈ 3.65 TB
```

| Metric | Value |
|--------|-------|
| Write QPS | ~231 ops/sec |
| Read QPS | ~2,310 ops/sec |
| Total Records (5 yrs) | ~36.5 Billion |
| Storage Required | ~3.65 TB |

> 💡 **Real-world ref**: [Bit.ly](https://bit.ly) handles billions of clicks per month. Their infrastructure uses distributed caching (Redis) and sharded databases to serve ~6B clicks/month efficiently.

---

## 4. API Design

Two core endpoints power the entire service.

### 🔵 POST — Generate Short URL

```http
POST www.bit.ly/api/v1/generate
Content-Type: application/json

Body:
{
  "longURL": "https://www.example.com/api/v1/userId=124"
}

Response:
{
  "shortURL": "www.bit.ly/3IY"
}
```

### 🟢 GET — Redirect to Long URL

```http
GET www.bit.ly/{shortURL}

Example:
GET www.bit.ly/3IY

Response: 302 Redirect → https://www.example.com/api/v1/userId=124
```

---

## 5. Redirection — 301 vs 302

> 🔑 This is a **classic interview trap**. Know the difference deeply.

### HTTP Status Code Comparison

| Code | Name | Behaviour | Caching | Server Load |
|------|------|-----------|---------|-------------|
| `301` | Permanent Redirect | Browser caches forever | ✅ Client-side cached | 🔻 Minimal DB hits |
| `302` | Temporary Redirect | No cache, hits server each time | ❌ Always hits server | ✅ Trackable |

### Flow Diagram

```
301 Flow (Cached):
Client ──► bit.ly/QW123 ──► [First visit: Server → DB lookup]
                         ──► Browser caches {bit.ly/QW123 → longURL}
                         ──► [Subsequent visits: skip server entirely]

302 Flow (Trackable):
Client ──► bit.ly/QW123 ──► Server ──► DB ──► 302 + longURL
                        ──► [Every visit tracked, analytics possible]
```

### ✅ Which to Use?

> **Use `302`** for production URL shorteners.

**Why?**
- Enables **user metrics** (click tracking, geography, device type)
- Supports **A/B testing** on redirects
- Keeps traffic flowing through your service (revenue/analytics)
- **[Bit.ly](https://bit.ly)**, **[TinyURL](https://tinyurl.com)**, **[Rebrandly](https://rebrandly.com)** all use `302`

---

## 6. Initial Flow Design

### URL Shortening Flow (POST)

```
client ──[POST longURL]──► Server ──► Hash Function ──► shortURL
                                          │
                                     HashMap / DB
                                     Key:   shortURL
                                     Value: longURL
```

**Steps:**
1. User sends a `POST` request with the `longURL`
2. Server runs it through a **Hash Function** (MD5/SHA-1/SHA-256)
3. Generated `shortURL` stored as **key**, `longURL` as **value**
4. `shortURL` returned to user

---

### URL Redirection Flow (GET)

```
client ──[GET shortURL]──► Server ──► map.get(shortURL) ──► longURL
                                          │
                                     302 Redirect ──► Browser opens longURL
```

**Steps:**
1. User pastes `shortURL` in browser
2. Browser sends `GET` request to service
3. Server does `hashmap.get(shortURL)` → retrieves `longURL`
4. Server responds with `302` → browser redirects

---

## 7. Database Choice — NoSQL vs SQL

### Initial Thought: NoSQL (Amazon DynamoDB / MongoDB)

```
Hashmap in Memory ──► NoSQL (Key-Value Store)
  Key:   shortURL
  Value: longURL
```

**Why NoSQL seems natural:**
- Key-value access pattern
- Simple structure

---

### Better Choice: ✅ SQL

> **Switch to SQL for production.**

```sql
CREATE TABLE urls (
  id        BIGINT PRIMARY KEY,   -- Unique ID (used for Base62)
  shortURL  VARCHAR(10),
  longURL   TEXT
);

-- Index for fast lookup
CREATE INDEX idx_short ON urls(shortURL);
```

**Why SQL wins:**

| Feature | NoSQL | SQL ✅ |
|---------|-------|--------|
| Complex queries | ❌ Limited | ✅ Full support |
| Write speed | Slower | **Faster** |
| Analytics queries | Hard | **Easy** |
| User-level stats (`how many URLs/user?`) | Hard | **Easy JOIN** |
| Indexing | Basic | **Advanced** |

> 💡 **Real-world ref**: [Pinterest](https://medium.com/pinterest-engineering) migrated from NoSQL to MySQL shards for exactly this reason — analytics & complex queries became essential.

---

## 8. Short URL Generation Strategies

### 🔢 Calculating URL Length — How Many Characters?

**Available characters:** `0–9`, `a–z`, `A–Z` → **62 symbols**

```
URL length = n characters
Possible combinations = 62ⁿ

n=1 →          62 URLs
n=2 →       3,844 URLs
n=3 →     238,328 URLs
n=6 →  56,800,235,584 URLs  ← ~56 Billion ✅

Need: 36.5 Billion  →  n = 6 is sufficient
```

**Result:** `www.bit.ly/_ _ _ _ _ _` (6-character short URL)

---

### Strategy 1: Hashing (MD5 / SHA-1 / SHA-256)

```
Long URL ──► [Hash Fn] ──► long_hash_string
                            "233eqwe23432eqw23"
                                    │
                           Take first 6 chars
                                    │
                               "asfw34"  ← shortURL
```

**Problem: Collisions!**

```
Loop:
  shortURL = hash(longURL)
  if shortURL exists in DB:
      longURL = longURL + "salt"   ← keep modifying
      try again
  else:
      save to DB
```

**Disadvantages:**
- 📈 DB hits increase significantly
- 🐢 Slow — recursive calls on collision
- ⚡ Not suitable for high-QPS systems

---

### Strategy 2: ✅ Base62 Encoding (Recommended)

> **This is the industry-standard approach.**

#### Base62 Conversion Table

| Base 10 | Base 62 |
|---------|---------|
| 0 | `0` |
| 9 | `9` |
| 10 | `a` |
| 35 | `z` |
| 36 | `A` |
| 61 | `Z` |

#### Example: Convert `14320` to Base62

```
14320 ÷ 62 = 230 remainder 60  →  'Y'
  230 ÷ 62 =   3 remainder 44  →  'I'
    3 ÷ 62 =   0 remainder  3  →  '3'

Read remainders bottom-up:  3IY
```

```
Long URL: www.example.com
Unique ID: 14320
Short URL: www.bit.ly/3IY   ✅
```

**SQL Record:**
```
| id    | shortURL | longURL             |
|-------|----------|---------------------|
| 14320 | 3IY      | www.example.com     |
```

#### Advantages vs Disadvantages

| | Base62 ✅ |
|-|-----------|
| ✅ Collision-free | Always unique (ID is unique → encoded value is unique) |
| ✅ Fast | No recursive lookup needed |
| ❌ Needs unique ID generator | Distributed systems need coordination |
| ❌ Variable length | ID=1 → `1` (1 char), ID=14320 → `3IY` (3 chars) |
| ❌ Guessable | Sequential IDs → next URL is predictable |

> 💡 **Real-world ref**: [YouTube](https://youtube.com) uses Base64 encoding for video IDs (e.g., `dQw4w9WgXcQ`). [Instagram](https://instagram.com) uses Base62 for photo IDs.

---

## 9. Distributed Architecture

### The Core Challenge

> In a distributed system with **multiple DB shards**, each shard generates its own IDs — **two shards can generate the same ID!**

**Solution: Centralized Unique ID Generator**

```
                  ┌─────────────────────┐
                  │  Unique ID Generator │  ← Centralized
                  │   (e.g. Snowflake)   │
                  └──────────┬──────────┘
                             │ unique_id
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          Server 1       Server 2       Server 3
              │              │              │
         base62(id)     base62(id)     base62(id)
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                    ┌────────────────┐
                    │   SQL DB (Sharded) │
                    └────────────────┘
```

### Complete System Architecture

#### Storing (POST) Flow
```
                              Load Balancer
Client ──[POST longURL]──►  ──────────────  ──► Application Server
                                                        │
                                            ┌───────────┴───────────┐
                                            │                       │
                                          Cache                     DB
                                       (Redis)               (SQL Shards)
                                            │
                                    Check if longURL
                                    already exists
                                    (avoid duplicates)
```

#### Redirection (GET) Flow
```
                              Load Balancer
Client ──[GET shortURL]──►  ──────────────  ──► Application Server
                                                        │
                                            ┌───────────┴───────────┐
                                            │                       │
                                          Cache ◄──── HIT?          DB
                                       (Redis)     MISS ────────────►
                                            │
                                      302 + longURL
                                            │
                                    Client redirects
```

> 💡 **Cache Strategy**: Cache hot short URLs (most clicked). Use **LRU eviction**. [Cloudflare](https://cloudflare.com) reports ~80% of traffic served from cache for URL services.

---

### Maximum Short URL Length — Why 6 is Enough

```
Our need:    36.5 Billion records
62^6 gives:  56  Billion combinations

56 Billion > 36.5 Billion  ✅

Unique ID will NEVER exceed 56 Billion within 5 years
→ Base62 result will ALWAYS be ≤ 6 characters
```

---

## 10. Key Takeaways

> The **mental model** you need to ace this in interviews.

| Concept | Decision | Reason |
|---------|----------|--------|
| Redirect code | **302** | Analytics & tracking |
| Database | **SQL** | Complex queries, better write speed |
| ID generation | **Base62** | Collision-free, fast |
| Caching | **Redis (LRU)** | Reduce DB reads by ~80% |
| URL length | **6 chars** | 56B combos > 36.5B needed |
| ID coordination | **Centralized generator** | Avoid duplicate IDs in shards |

---

### 🏗️ Professional Systems That Use These Patterns

| Company | What They Do | Pattern Used |
|---------|-------------|--------------|
| [Bit.ly](https://bit.ly) | URL Shortener | Base62, 302, Redis |
| [TinyURL](https://tinyurl.com) | URL Shortener | Hashing + collision handling |
| [YouTube](https://youtube.com) | Video IDs | Base64 encoding |
| [Instagram](https://instagram.com) | Photo IDs | Base62 + Snowflake IDs |
| [Twitter/X](https://twitter.com) | Tweet IDs | Snowflake (distributed ID gen) |

---

### 📊 Interview Checklist

```
[ ] Ask clarifying questions (scale, constraints, characters)
[ ] Do back-of-the-envelope estimation (QPS, storage)
[ ] Design API endpoints (POST generate, GET redirect)
[ ] Decide redirect type (301 vs 302 — always 302)
[ ] Choose DB (SQL > NoSQL for this use case)
[ ] Decide URL generation strategy (Base62 > Hashing)
[ ] Plan for distributed system (sharding, ID gen, cache)
[ ] Mention cache layer (Redis, LRU eviction)
```

---

*Notes from HLD Lecture 6 · URL Shortener (Bit.ly Clone) · System Design*
