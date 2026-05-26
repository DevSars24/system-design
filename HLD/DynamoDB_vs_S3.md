# 🗄️ DynamoDB vs S3 — Where Should Data Live?

> **Core Question:** Both store data — but they are optimized for fundamentally different problems.  
> Choosing the wrong one costs money *and* performance.

---

## 📖 Table of Contents

1. [The Core Difference](#1-the-core-difference)
2. [DynamoDB — Optimized for Access](#2-dynamodb--optimized-for-access)
3. [S3 — Optimized for Durable Storage](#3-s3--optimized-for-durable-storage)
4. [Data Lifecycle — The Real Decision Driver](#4-data-lifecycle--the-real-decision-driver)
5. [What S3 Powers at Internet Scale](#5-what-s3-powers-at-internet-scale)
6. [Side-by-Side Comparison](#6-side-by-side-comparison)
7. [Decision Guide](#7-decision-guide)
8. [Quick Revision Cheatsheet](#8-quick-revision-cheatsheet)

---

## 1. The Core Difference

```mermaid
graph TD
    D[Data Storage Decision] --> DDB[DynamoDB]
    D --> S3[Amazon S3]

    DDB --> DDB1[Key-Value Database]
    DDB --> DDB2[Millisecond latency]
    DDB --> DDB3[Lives on the hot request path]
    DDB --> DDB4[Cost scales with reads and writes]

    S3 --> S31[Object Storage]
    S3 --> S32[Higher latency but much cheaper]
    S3 --> S33[Durable, virtually unlimited scale]
    S3 --> S34[Cost scales with data stored]
```

> 💡 Both store data — but DynamoDB is a **database** built for access speed.  
> S3 is a **storage system** built for durability and scale.

---

## 2. DynamoDB — Optimized for Access

> **Built for:** Predictable, low-latency access to data at massive scale.

```mermaid
sequenceDiagram
    participant App as Application
    participant DDB as DynamoDB

    App->>DDB: GET user:42 (partition key)
    Note over DDB: Route directly to correct partition
    DDB-->>App: ⚡ Response in < 10ms
    Note over App,DDB: Consistent latency regardless of total scale
```

### How DynamoDB Routes Requests

```mermaid
graph LR
    RQ[Request: GET post_id=999] --> HASH[Hash partition key]
    HASH --> PART[Route to correct partition]
    PART --> NODE[Read from that node]
    NODE --> RESP[Return data ⚡]
```

| Characteristic | Detail |
|---|---|
| **Access pattern** | Key-value lookup by partition key |
| **Latency** | Single-digit milliseconds |
| **Pricing driver** | Read/Write Capacity Units (throughput) |
| **Best for** | Live app data sitting on the hot request path |
| **Table classes** | Standard (hot data) · Standard-IA (infrequent access) |
| **Max item size** | 400 KB per item |

> ⚠️ **Cost Warning:** DynamoDB Standard-IA is cheaper for storage but read costs are higher.  
> If your data is so cold you barely access it — it probably shouldn't be in DynamoDB at all.

---

## 3. S3 — Optimized for Durable Storage

> **Built for:** Storing massive amounts of data cheaply with extremely high durability.

```mermaid
sequenceDiagram
    participant App as Application
    participant S3 as Amazon S3
    participant CDN as CloudFront CDN

    App->>S3: PUT image.jpg (store once)
    S3-->>App: ✅ Stored (11 nines durability)

    Note over S3,CDN: Serve to millions of users via CDN
    CDN->>S3: Origin fetch on first cache miss
    S3-->>CDN: File content
    CDN-->>App: Cached & fast delivery ⚡
```

### S3 Storage Classes (Cost vs Speed)

```mermaid
graph TD
    S3[S3 Storage Classes] --> ST[S3 Standard\nFrequent access]
    S3 --> IA[S3 Standard-IA\nInfrequent access]
    S3 --> OZ[S3 One Zone-IA\nLower redundancy]
    S3 --> GL[S3 Glacier\nArchival - minutes to hours]
    S3 --> GD[S3 Glacier Deep Archive\nCheapest - 12hr retrieval]

    ST --> |"Most expensive\nLowest latency"| ST2[Hot assets\nImages, JS bundles]
    GL --> |"Cheapest\nHighest retrieval latency"| GL2[Compliance archives\nOld logs]
```

| Characteristic | Detail |
|---|---|
| **Access pattern** | Object key lookup (not query-able like SQL) |
| **Latency** | Tens to hundreds of milliseconds |
| **Pricing driver** | GB stored + per-request cost |
| **Best for** | Static assets, archives, data lakes, backups |
| **Durability** | 99.999999999% (11 nines) |
| **Max object size** | 5 TB per object |

---

## 4. Data Lifecycle — The Real Decision Driver

> Most data **doesn't stay hot forever**. It follows a lifecycle —  
> and the right storage system depends on **where in the lifecycle it is**.

```mermaid
flowchart LR
    A["🔥 Hot Phase\nActive reads & writes"] -->|Aging out| B["🌡️ Warm Phase\nLess frequent access"]
    B -->|Rarely accessed| C["🧊 Cold Phase\nArchival / reference only"]

    A --> DDB1["DynamoDB Standard\n(fast, expensive)"]
    B --> DDB2["DynamoDB Standard-IA\n(cheaper, still queryable)"]
    C --> S3_A["S3 Standard or S3 Glacier\n(cheapest, durable)"]
```

### Real-World Example — Messaging System

```mermaid
sequenceDiagram
    participant U as User
    participant App as App Server
    participant DDB as DynamoDB
    participant S3 as Amazon S3

    U->>App: Open chat
    App->>DDB: Fetch last 30 messages (hot)
    DDB-->>App: ⚡ Returned in ms
    App-->>U: Display conversation

    Note over DDB,S3: Background job runs nightly
    DDB->>S3: Export messages older than 90 days
    S3-->>S3: Stored durably at low cost 💾

    U->>App: Load history from 2023
    App->>S3: Fetch archived messages
    S3-->>App: 📦 Retrieved (higher latency is OK here)
    App-->>U: Historical view loaded
```

> 💡 **Pattern:** Keep recent data in DynamoDB for speed.  
> Offload aged data to S3 for cost. Queries on archived data are rare — latency is acceptable.

---

## 5. What S3 Powers at Internet Scale

```mermaid
mindmap
  root((Amazon S3\nUse Cases))
    Static Assets
      Website HTML CSS JS
      Frontend build bundles
      Images and videos
    Media Delivery
      CDN origin layer
      Video streaming
      Software downloads
    Data Engineering
      Data lakes
      Analytics datasets
      ML training data
    Operational
      Database backups
      Log archives
      Disaster recovery snapshots
    ML Artifacts
      Model weights
      Feature stores
      Experiment snapshots
```

> S3 often acts as the **backbone storage layer** for large portions of the internet —  
> not because it's fast, but because it's **cheap, durable, and infinitely scalable**.

---

## 6. Side-by-Side Comparison

```mermaid
quadrantChart
    title DynamoDB vs S3 — Access Speed vs Storage Cost
    x-axis High Storage Cost --> Low Storage Cost
    y-axis High Latency --> Low Latency
    quadrant-1 Expensive but fast
    quadrant-2 Best of both
    quadrant-3 Avoid
    quadrant-4 Cheap and durable
    DynamoDB Standard: [0.15, 0.90]
    DynamoDB Standard-IA: [0.30, 0.85]
    S3 Standard: [0.75, 0.40]
    S3 Glacier: [0.95, 0.10]
```

| Dimension | DynamoDB | S3 |
|---|---|---|
| **Primary abstraction** | Table → Items → Attributes | Bucket → Objects |
| **Query capability** | ✅ Rich (GSI, LSI, filters) | ❌ None (key lookup only) |
| **Latency** | ⚡ Single-digit ms | 🐢 Tens–hundreds of ms |
| **Cost model** | Per read/write unit | Per GB stored |
| **Max item/object size** | 400 KB per item | 5 TB per object |
| **Auto-scaling** | ✅ Provisioned or On-Demand | ✅ Unlimited objects |
| **Durability** | ✅ Multi-AZ replication | ✅ 11 nines |
| **CDN integration** | ❌ Not designed for it | ✅ Native with CloudFront |
| **Best on request path** | ✅ Yes | ⚠️ Possible but not ideal |
| **Best for large files** | ❌ 400KB limit | ✅ Up to 5TB |

---

## 7. Decision Guide

```mermaid
flowchart TD
    START[Where should this data live?] --> Q1{Is it on the hot\nrequest path?}
    Q1 -->|Yes — needs ms latency| Q2{Structured key-value data?}
    Q1 -->|No — batch or archival| Q3{Large blob or file?}

    Q2 -->|Yes| DDB[DynamoDB ✅]
    Q2 -->|No, needs SQL joins| RDS[RDS / Aurora]

    Q3 -->|Yes — image, video, doc| S3[S3 ✅]
    Q3 -->|No — rarely queried records| Q4{How old is the data?}

    Q4 -->|Under 90 days| DDBIA[DynamoDB Standard-IA]
    Q4 -->|Older — archival| S3_G[S3 Standard or S3 Glacier]
```

> 💡 **Rule of thumb:**
> - Data your app reads **per user request** → **DynamoDB**
> - Data that just needs to **exist reliably and cheaply** → **S3**
> - Data that **started hot but aged out** → migrate DynamoDB → S3

---

## 8. Quick Revision Cheatsheet

```mermaid
mindmap
  root((DynamoDB\nvs S3))
    DynamoDB
      Key-value database
      Single-digit ms latency
      Priced per read and write
      Hot request path data
      Standard vs Standard-IA table classes
      400KB max item size
    S3
      Object storage
      Tens to hundreds of ms latency
      Priced per GB stored
      Durable 11 nines
      Unlimited scale
      5TB max object size
      CDN origin layer
    Data Lifecycle
      Hot = DynamoDB Standard
      Warm = DynamoDB Standard-IA
      Cold = S3 Standard
      Archival = S3 Glacier
    Key Decision
      Request path = DynamoDB
      Just needs to exist = S3
      Large files = always S3
      Aged out data = migrate to S3
```

### ⚡ One-Liner Summaries

| Concept | One-Line Summary |
|---|---|
| **DynamoDB** | Key-value DB for ms-latency hot request path data — priced per read/write |
| **S3** | Object store for durable, cheap, scalable storage — priced per GB stored |
| **DynamoDB Standard-IA** | Same as Standard but cheaper storage, higher read cost — for cold table data |
| **S3 Glacier** | Cheapest long-term archive — retrieval takes minutes to hours |
| **Data lifecycle** | Hot → DynamoDB → DynamoDB-IA → S3 → S3 Glacier as data ages |
| **S3 + CDN** | S3 as origin + CloudFront = serve static assets at internet scale |
| **The real question** | Not DynamoDB vs S3 — it's *what phase of lifecycle is this data in?* |

---

> 📅 **Notes compiled from:** DynamoDB vs S3: Rethinking Where Data Should Live — Aasma Garg
