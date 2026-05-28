# 🗄️ HLD Lecture 15 — SQL Architecture Deep Dive

> **From raw rows to blazing-fast queries — understanding the full stack of relational databases**

---

## 📑 Table of Contents

1. [SQL / Relational Architecture](#1-sql--relational-architecture)
2. [Data Storage Internals](#2-data-storage-internals)
3. [Master-Slave Architecture](#3-master-slave-architecture)
4. [Horizontal Scaling → Sharding](#4-horizontal-scaling--sharding)
5. [Indexing Basics](#5-indexing-basics)
6. [B-Trees](#6-b-trees)
7. [B+ Trees](#7-b-trees-1)
8. [Clustered vs Non-Clustered Indexing](#8-clustered-vs-non-clustered-indexing)
9. [NoSQL Comparison](#9-nosql-comparison)
10. [Quick Summary Cheatsheet](#10-quick-summary-cheatsheet)

---

## 1. SQL / Relational Architecture

### 🏗️ What is a Relational Database?

A **Relational Database** organizes data into **rows and columns** (just like an Excel sheet, but way more powerful). It is a **logical arrangement** — the table you see in a query is **not** physically stored that way on disk.

```
📋 Logical Layer  → What you SEE (Tables, Views, Rows, Columns)
💾 Physical Layer → What's on DISK (Sectors, Data Blocks, Data Pages)
```

### 🍕 Real-World Analogy: Restaurant Order System

| userId | name   | email       |
|--------|--------|-------------|
| 101    | Aditya | xyz@mail.com |
| 102    | Rohit  | ert@mail.com |
| 103    | Mukesh | wer@mail.com |

Think of this table like a **restaurant's order book**:
- Each **row** = one customer's order
- Each **column** = a specific detail (name, table number, food item)
- The **DBMS** is the waiter who finds your order when you call

### 🔭 Views → Virtual Tables

```sql
-- A VIEW is like a filtered window into your data
CREATE VIEW active_users AS
  SELECT userId, name FROM user WHERE active = 1;
-- No data is actually duplicated — it's just a saved query!
```

> 💡 **Think of Views** like a Snapchat filter — the original face (data) is unchanged, but you see a custom version.

---

## 2. Data Storage Internals

### 🧱 The Physical Storage Hierarchy

```mermaid
graph TD
    A["💽 Hard Disk"] --> B["📦 Sector\n(smallest hardware unit)"]
    B --> C["🧱 Data Block\n8KB - 64KB\n(smallest I/O unit)"]
    C --> D["📄 Data Page\n8KB = 8192 bytes\n(managed by DBMS)"]
    D --> E["📝 Data Records\n(actual rows of your table)"]

    style A fill:#1a1a2e,color:#e0e0e0,stroke:#4a90d9
    style B fill:#16213e,color:#e0e0e0,stroke:#4a90d9
    style C fill:#0f3460,color:#e0e0e0,stroke:#4fc3f7
    style D fill:#533483,color:#e0e0e0,stroke:#ce93d8
    style E fill:#2d6a4f,color:#e0e0e0,stroke:#74c69d
```

### 📄 Anatomy of a Data Page

```mermaid
block-beta
  columns 3
  A["Page Header\n(96 bytes)\nPage Number, Checksum"]:1
  B["Data Records\n(8060 bytes)\nActual row data"]:1
  C["Row Offset Array\n(36 bytes)\nPointer to each row"]:1
```

```
┌─────────────────────────────────────────────┐
│           DATA PAGE  (8 KB = 8192 bytes)    │
├──────────────┬──────────────────┬───────────┤
│ Page Header  │   Data Records   │  Offset   │
│   96 bytes   │   8060 bytes     │ 36 bytes  │
│ (page no,    │ (your actual     │ (array of │
│  checksum)   │  row data)       │  row ptrs)│
└──────────────┴──────────────────┴───────────┘
```

### 🔢 Real-World Math Breakdown

Imagine a **User table with 1 Million rows**:

| Parameter | Value | Explanation |
|-----------|-------|-------------|
| 1 Data Page | 8 KB = 8192 bytes | Standard SQL Server page size |
| Page Header | 96 bytes | Overhead for page metadata |
| Usable space | 8060 bytes | For actual row data |
| Row offset | 36 bytes | Fixed overhead per page |
| Avg row size | **32 bytes** | userId(4) + name(16) + email(12) |
| Rows per page | **~251 rows** | 8060 / 32 ≈ 251 |
| Total pages needed | **~3,984 pages** | 1,000,000 / 251 |
| Total data blocks | **~1,992 blocks** | 3,984 / 2 (2 pages per 8KB block) |

### 🗺️ DBMS Mapping Table

The DBMS keeps a **mapping table** to know which data page lives in which data block:

```
DBMS Mapping Table:
├── Data page 1    ──►  Data Block 1
├── Data page 2    ──►  Data Block 1
├── Data page 3    ──►  Data Block 2
│   ...
└── Data page 3984 ──►  Data Block 1992
```

### 🐌 The Performance Problem (Without Indexing)

```sql
SELECT * FROM user WHERE userId = 101;
```

```mermaid
sequenceDiagram
    participant App as 🖥️ Application
    participant DBMS as 🧠 DBMS
    participant Disk as 💽 Hard Disk

    App->>DBMS: SELECT * FROM user WHERE userId = 101
    DBMS->>Disk: Load Data Block 1 (pages 1,2)
    Disk-->>DBMS: Return rows - not found
    DBMS->>Disk: Load Data Block 2 (pages 3,4)
    Disk-->>DBMS: Return rows - not found
    DBMS->>Disk: Load Data Block 3...
    Note over DBMS,Disk: 😱 1,992 blocks to check!
    Note over App,Disk: O(N) — Scans ALL blocks!
    Disk-->>DBMS: Finally found userId=101!
    DBMS-->>App: Return result
```

> ⚠️ **The Problem**: Without indexing, every query does a **Full Table Scan** — O(N) complexity. With 10 million rows, this reads millions of data blocks!

---

## 3. Master-Slave Architecture

### 🎯 The Problem It Solves

Your app grows. Suddenly you have **millions of users** hitting your DB. A single database server **can't handle it all**!

### 🏛️ Solution: Master-Slave (Primary-Replica)

```mermaid
graph TD
    S["🖥️ Application Server"] -->|"✍️ WRITE\n(INSERT, UPDATE, DELETE)"| M

    M["👑 MASTER DB\n(Primary)"]

    M -->|"🔄 Async Replication"| S1["🔵 SLAVE 1\n(Replica)"]
    M -->|"🔄 Async Replication"| S2["🟢 SLAVE 2\n(Replica)"]
    M -->|"🔄 Async Replication"| S3["🟡 SLAVE 3\n(Replica)"]

    S -->|"📖 READ"| S1
    S -->|"📖 READ"| S2
    S -->|"📖 READ"| S3

    style M fill:#c0392b,color:#fff,stroke:#e74c3c
    style S1 fill:#2980b9,color:#fff,stroke:#3498db
    style S2 fill:#27ae60,color:#fff,stroke:#2ecc71
    style S3 fill:#f39c12,color:#fff,stroke:#f1c40f
    style S fill:#6c3483,color:#fff,stroke:#9b59b6
```

### 📋 Master vs Slave — Quick Rules

| Feature | Master (Primary) | Slave (Replica) |
|---------|-----------------|-----------------|
| **Writes** | ✅ Yes | ❌ No |
| **Reads** | ✅ Yes (avoid) | ✅ Yes (preferred) |
| **Count** | 1 per cluster | Multiple |
| **Failover** | Promoted from slave | Promoted to master |
| **Purpose** | Handle all writes | Scale reads horizontally |

### 🍎 Real-World: Netflix

> Netflix uses **Master-Slave** for its user preference data. When you pause a show, the **master** gets the write. When 80 million users browse the homepage simultaneously, **read replicas** serve all those catalogue reads — the master is untouched.

---

## 4. Horizontal Scaling → Sharding

### 🤔 Master-Slave isn't enough

What if your **write load** itself becomes too heavy? You can't have more than one master in basic replication. Enter: **Sharding**!

### 🧩 Sharding: Split the Data Across Multiple DBs

```mermaid
graph TD
    APP["🖥️ Application"] --> LB["⚖️ Shard Router\n(Which shard?)"]

    LB -->|"userId 1–10,000"| S1["🗄️ Shard 1\nuserId: 1–10,001"]
    LB -->|"userId 10,001–20,000"| S2["🗄️ Shard 2\nuserId: 10,001–20,000"]
    LB -->|"userId 20,001+"| S3["🗄️ Shard 3\nuserId: 20,001+"]

    S1 --> R1["🔵 Replica 1A\n🔵 Replica 1B"]
    S2 --> R2["🟢 Replica 2A\n🟢 Replica 2B"]
    S3 --> R3["🟡 Replica 3A\n🟡 Replica 3B"]

    style APP fill:#6c3483,color:#fff
    style LB fill:#1a5276,color:#fff
    style S1 fill:#0f3460,color:#fff
    style S2 fill:#0f3460,color:#fff
    style S3 fill:#0f3460,color:#fff
```

### 🍕 Pizza Analogy

> You're running a **pizza chain**. One store (master) can't serve all of Mumbai. So you open **3 stores** (shards):
> - **Shard 1**: Customers in Andheri → orders go to Andheri store
> - **Shard 2**: Customers in Bandra → orders go to Bandra store
> - **Shard 3**: Customers in Powai → orders go to Powai store

Each store is independent — horizontal scaling!

---

## 5. Indexing Basics

### 🐢 The Problem

```sql
-- 10 million users, no index:
SELECT * FROM user WHERE userId = 5000000;
-- Scans ALL 3,984+ pages = SLOW 🐌
```

Without indexing → **O(N)** — reads every single data block.

### 🚀 The Solution: Indexing

```mermaid
graph LR
    A["❓ Query:\nuserId = 101"] --> B["🗂️ Index\n(B+ Tree)"]
    B --> C["🎯 Exact page: \nData Page 1"]
    C --> D["✅ Found!\nO(log N)"]

    A2["❓ Without Index"] --> B2["🔁 Full Scan:\n1,992 data blocks"]
    B2 --> D2["😱 O(N)"]

    style B fill:#155724,color:#fff
    style D fill:#155724,color:#fff
    style B2 fill:#721c24,color:#fff
    style D2 fill:#721c24,color:#fff
```

| Without Index | With Index |
|--------------|-----------|
| O(N) | O(log N) |
| Scans all blocks | Jumps directly to page |
| Slow on large tables | Fast even at 100M rows |
| Full table scan | Index seek |

### 🗂️ Types of Indexing

```mermaid
mindmap
  root((Indexing))
    Clustered
      Leaf nodes store FULL row data
      Only 1 per table allowed
      Data physically sorted by index key
      Super fast for range queries
    Non-Clustered
      Leaf nodes store only row ID/page no
      Multiple allowed per table
      Separate structure from actual data
      Extra lookup step needed
```

---

## 6. B-Trees

### 🌳 The DSA Behind Indexing

Before B-Trees, let's understand the evolution:

```mermaid
graph LR
    BST["🌲 BST\nBinary Search Tree\n2 children max"] --> TST["🌳 TST\nTernary Search Tree\n3 children max"]
    TST --> MW["🌴 M-Way Tree\nM children max\nk = M-1 keys per node"]
    MW --> BT["🔵 B-Tree\nBalanced M-way\nAuto-rebalances"]

    style BT fill:#1a5276,color:#fff
    style MW fill:#0f3460,color:#fff
```

### 🔵 B-Tree Structure (M = 3)

Rules for a B-Tree with **order M = 3**:
- Each node has **at most M children** = 3
- Each node has **at most M-1 keys** = 2 keys
- The tree is **always balanced** (auto-rebalances on insert)

```mermaid
graph TD
    Root["[5 | 9]\n Root Node"] -->|"< 5"| L1["[3]"]
    Root -->|"5-9"| L2["[7]"]
    Root -->|"> 9"| L3["[11 | 23]"]

    L1 --> LL1["[1 | 2]"]
    L1 --> LL2["[4]"]
    L2 --> LL3["[6]"]
    L2 --> LL4["[8]"]
    L3 --> LL5["[10]"]
    L3 --> LL6["[16]"]
    L3 --> LL7["[25]"]

    style Root fill:#1a5276,color:#fff,stroke:#4a90d9
    style L1 fill:#0f3460,color:#fff
    style L2 fill:#0f3460,color:#fff
    style L3 fill:#0f3460,color:#fff
```

### 🔄 B-Tree Split (Insertion Process)

When inserting **9, 11, 7, 3, 10, 23, 5, 16, 12, 34, 1, 89**:

```mermaid
graph TD
    subgraph Step1["Step 1: Insert 9, 11 → Node is [9|11]"]
        A1["[9 | 11]"]
    end

    subgraph Step2["Step 2: Insert 7 → overflow! SPLIT!"]
        B1["[9]"]
        B2["[7]"] --> B1
        B3["[11]"] --> B1
    end

    subgraph Step3["Step 3: More inserts → Tree grows down"]
        C1["[9 | 19]"]
        C2["[3 | 7]"] --> C1
        C3["[10 | 11]"] --> C1
        C4["[23]"] --> C1
    end

    style B1 fill:#c0392b,color:#fff
    style C1 fill:#1a5276,color:#fff
```

### 💾 B-Tree Node Code Structure

```c
struct BTreeNode {
    int keys[M-1];         // e.g., [5, 9] for M=3
    BTreeNode* children[M]; // pointers to children
    int keyCount;           // current number of keys
    bool isLeaf;
};
```

---

## 7. B+ Trees

### 🌟 Why B+ Trees? The Upgrade from B-Trees

B-Trees are good, but **databases specifically use B+ Trees** for a key reason:

```mermaid
graph LR
    subgraph BTree["B-Tree"]
        BT_Root["[5 | 9]"]
        BT_L["[3]"] --> BT_Root
        BT_R["[11]"] --> BT_Root
        Note1["❌ Data stored in ALL nodes\n❌ Range queries are hard\n(must traverse up and down)"]
    end

    subgraph BPlusTree["B+ Tree"]
        BP_Root["[5 | 9]"]
        BP_L["[3 | 4]"] --> BP_Root
        BP_M["[6 | 7 | 8]"] --> BP_Root
        BP_R["[10 | 11]"] --> BP_Root
        Note2["✅ Data ONLY in leaf nodes\n✅ Leaf nodes linked together\n✅ Range queries = walk the list!"]
        BP_L -.->|"linked"| BP_M -.->|"linked"| BP_R
    end
```

### 🔑 Key Differences: B-Tree vs B+ Tree

| Feature | B-Tree | B+ Tree |
|---------|--------|---------|
| Data location | All nodes | **Leaf nodes only** |
| Intermediate nodes | Store data + keys | Store keys only (as guides) |
| Leaf node links | ❌ Not linked | ✅ **Linked list** |
| Range queries | Slow (backtrack) | **Fast** (traverse leaf list) |
| Used in DBs | Rarely | **YES — default** |

### 🔵 B+ Tree Visual

```mermaid
graph TD
    Root["🗂️ [9 | 10]\n(Guide keys only, NO data)"]

    Root -->|"< 9"| L1["🍃 [4 | 7 | 8]"]
    Root -->|"9-10"| L2["🍃 [9 | 10]"]
    Root -->|"> 10"| L3["🍃 [11 | 12...]"]

    L1 -.->|"🔗 Linked List"| L2
    L2 -.->|"🔗 Linked List"| L3

    L1 --- D1["📄 Actual Data/Page Ptr"]
    L2 --- D2["📄 Actual Data/Page Ptr"]
    L3 --- D3["📄 Actual Data/Page Ptr"]

    style Root fill:#1a5276,color:#fff,stroke:#4fc3f7
    style L1 fill:#155724,color:#fff,stroke:#74c69d
    style L2 fill:#155724,color:#fff,stroke:#74c69d
    style L3 fill:#155724,color:#fff,stroke:#74c69d
```

### ⚡ Range Query Power

```sql
-- Range query: userId between 101 and 205
SELECT * FROM user WHERE userId BETWEEN 101 AND 205;
```

```mermaid
sequenceDiagram
    participant Q as 📝 Query
    participant BPT as 🌲 B+ Tree
    participant L1 as 🍃 Leaf: [101-150]
    participant L2 as 🍃 Leaf: [151-200]
    participant L3 as 🍃 Leaf: [201-205]

    Q->>BPT: Find userId = 101 (start)
    BPT->>L1: O(log N) → Found page for 101!
    Note over L1,L3: Now just WALK the linked list →
    L1->>L2: Next leaf (linked!)
    L2->>L3: Next leaf (linked!)
    L3-->>Q: Done! Range [101-205] fetched efficiently ⚡
```

> 🎯 **B+ Tree is perfect for range queries** because leaf nodes form a doubly-linked list — no tree traversal needed after finding the start!

---

## 8. Clustered vs Non-Clustered Indexing

### 🏠 Clustered Index — The "Living Together" Index

> 🏠 **Analogy**: In a Clustered Index, the data IS the index. Think of a **physical library** where books are arranged alphabetically ON THE SHELF. The arrangement IS the index!

```mermaid
graph TD
    subgraph CI["🟢 Clustered Index (B+ Tree)"]
        CRoot["Root: [19 | 25]"]
        CL1["[1 | 6]"] --> CRoot
        CL2["[17 | 19]"] --> CRoot
        CL3["[25 | 30]"] --> CRoot

        CL1 -->|"Full row data!"| CD1["📄 Page 1\n1, Alice, alice@mail.com\n6, Bob, bob@mail.com"]
        CL2 -->|"Full row data!"| CD2["📄 Page 2\n17, Charlie, c@mail.com\n19, Aditya, xyz@mail.com"]
        CL3 -->|"Full row data!"| CD3["📄 Page 3\n25, Rohit, ert@mail.com\n30, Mukesh, wer@mail.com"]

        CD1 -.->|"linked"| CD2 -.->|"linked"| CD3
    end

    style CI fill:#0d1b0d,stroke:#74c69d
    style CD1 fill:#155724,color:#fff
    style CD2 fill:#155724,color:#fff
    style CD3 fill:#155724,color:#fff
```

### 📑 Non-Clustered Index — The "Lookup Table" Index

> 📑 **Analogy**: Non-Clustered Index is like the **index at the back of a textbook** — it tells you "B+ Trees → see page 47" but the actual content is on page 47, not in the index itself.

```mermaid
graph TD
    subgraph NCI["🔵 Non-Clustered Index (B+ Tree on 'name')"]
        NRoot["Root: [M | R]"]
        NL1["[Aditya | Bob]"] --> NRoot
        NL2["[Mukesh | Nita]"] --> NRoot
        NL3["[Rohit | Zara]"] --> NRoot

        NL1 -->|"Only page number!"| ND1["📋 Aditya → Page 2, Row 4\nBob → Page 1, Row 2"]
        NL2 -->|"Only page number!"| ND2["📋 Mukesh → Page 3, Row 2\nNita → Page 5, Row 1"]
        NL3 -->|"Only page number!"| ND3["📋 Rohit → Page 3, Row 1\nZara → Page 8, Row 7"]
    end

    subgraph Extra["Extra lookup step needed!"]
        ND1 -->|"Go to actual page"| ACTUAL["📄 Actual Data Pages\n(Full row data lives here)"]
        ND2 --> ACTUAL
        ND3 --> ACTUAL
    end

    style NCI fill:#1a0d1a,stroke:#ce93d8
    style ND1 fill:#6c3483,color:#fff
    style ND2 fill:#6c3483,color:#fff
    style ND3 fill:#6c3483,color:#fff
    style ACTUAL fill:#1a5276,color:#fff
```

### ⚔️ Head-to-Head Comparison

| Feature | 🟢 Clustered Index | 🔵 Non-Clustered Index |
|---------|-------------------|----------------------|
| **Leaf nodes store** | **Full row data** | **Only page number / row ID** |
| **Count per table** | **Only 1** (data can only be sorted one way!) | **Multiple** (each on diff column) |
| **Speed (exact lookup)** | ⚡ Very fast | 🔄 Fast (extra pointer hop) |
| **Range queries** | ⚡⚡ Extremely fast | Slower (many pointer hops) |
| **INSERT speed** | 🐢 Slower (must maintain order) | Faster |
| **Storage** | Base table IS the index | Extra storage required |
| **Default for PK** | ✅ Yes (auto-created) | ❌ No |

### 🚨 Why Not Index Every Column?

```mermaid
graph LR
    Q["💡 'Why not index\nevery column?'"] --> A1["📦 Each index = extra B+ Tree"]
    A1 --> A2["💾 Huge space overhead\n(can be larger than table itself!)"]
    A2 --> A3["🐢 Slower writes\n(every INSERT updates all indexes)"]
    A3 --> A4["✅ Index strategically!\nOnly high-cardinality, frequently\nqueried columns"]

    style Q fill:#1a5276,color:#fff
    style A4 fill:#155724,color:#fff
    style A2 fill:#721c24,color:#fff
    style A3 fill:#721c24,color:#fff
```

### 🎯 Real-World: Which Columns to Index?

| Column | Index? | Why? |
|--------|--------|------|
| `userId` (Primary Key) | ✅ Auto clustered | Unique, frequently queried |
| `email` | ✅ Non-clustered | Login queries need fast lookup |
| `name` | ⚠️ Maybe | Depends on search frequency |
| `created_at` | ✅ Non-clustered | Range queries (date filters) |
| `gender` | ❌ No | Low cardinality (only M/F) — bad index |
| `bio_text` | ❌ No | Full-text search needed instead |

---

## 9. NoSQL Comparison

### 🗄️ How Different NoSQL DBs Handle Indexing

```mermaid
graph LR
    subgraph SQL["🔵 SQL Databases"]
        SQL_DB["PostgreSQL, MySQL,\nSQL Server, Oracle"]
        SQL_IDX["B+ Trees\n(Primary + Secondary\nIndexes)"]
        SQL_DB --> SQL_IDX
    end

    subgraph Mongo["🟢 MongoDB"]
        M_DB["Document Store\nJSON-like documents"]
        M_IDX["B+ Trees\n(Same as SQL!)"]
        M_DB --> M_IDX
    end

    subgraph Cassandra["🟡 Apache Cassandra"]
        C_DB["Wide-Column Store\nDistributed by design"]
        C_IDX["LSM Tree\n(Log-Structured\nMerge-Tree)"]
        C_DB --> C_IDX
    end

    style SQL_IDX fill:#1a5276,color:#fff
    style M_IDX fill:#155724,color:#fff
    style C_IDX fill:#7b3500,color:#fff
```

### 🌲 What is an LSM Tree? (Cassandra's Secret Weapon)

```mermaid
graph TD
    Write["✍️ Write Request"]
    Write --> Mem["⚡ MemTable\n(In-Memory, super fast)\nAll writes go here first!"]
    Mem -->|"When full, flush to disk"| SST1["📄 SSTable 1\n(Immutable, sorted file)"]
    SST1 --> SST2["📄 SSTable 2"]
    SST2 --> SST3["📄 SSTable 3"]

    SST1 & SST2 & SST3 -->|"Background Compaction"| Merged["📦 Merged & Sorted SSTable\n(Removes duplicates/tombstones)"]

    style Mem fill:#c0392b,color:#fff
    style SST1 fill:#1a5276,color:#fff
    style SST2 fill:#1a5276,color:#fff
    style SST3 fill:#1a5276,color:#fff
    style Merged fill:#155724,color:#fff
```

### ⚔️ B+ Tree vs LSM Tree

| Feature | B+ Tree (SQL, MongoDB) | LSM Tree (Cassandra) |
|---------|----------------------|---------------------|
| **Write speed** | 🔄 Moderate (update in-place) | ⚡ **Super fast** (append-only) |
| **Read speed** | ⚡ **Super fast** | 🔄 Moderate (check multiple files) |
| **Range queries** | ⚡ Excellent | 🔄 Good (after compaction) |
| **Best for** | Read-heavy workloads | Write-heavy workloads |
| **Space efficiency** | Good | Lower (until compaction) |
| **Real-world use** | E-commerce, Banking | IoT, Logging, Analytics |

### 🏢 Real-World Scenario

| Use Case | Recommended DB | Why |
|----------|---------------|-----|
| 🏦 Banking / Payments | PostgreSQL (B+ Tree) | Consistency + fast reads |
| 📱 Social Media Feed | MongoDB (B+ Tree) | Flexible schema + reads |
| 📊 IoT Sensor Data | Cassandra (LSM Tree) | 1M+ writes/sec |
| 🛒 E-commerce Catalog | MySQL (B+ Tree) | Complex queries + joins |
| 📝 Logging Pipeline | Cassandra / ClickHouse | Append-heavy writes |

---

## 10. Quick Summary Cheatsheet

```mermaid
mindmap
  root((SQL Architecture\nLEC 15))
    Data Storage
      Sector → Block → Page
      1 Data Page = 8KB ≈ 251 rows
      DBMS manages pages
      HDD manages blocks
    Master-Slave
      Master handles writes
      Slaves handle reads
      Async replication
    Sharding
      Horizontal partitioning
      Each shard = subset of data
      Range or Hash based
    Indexing
      B+ Tree based
      O log N lookup
      Turns full scan into index seek
    Clustered Index
      Leaf = full row data
      1 per table
      PK default
    Non-Clustered Index
      Leaf = page pointer only
      Multiple per table
      Extra lookup hop
    NoSQL
      MongoDB → B+ Tree
      Cassandra → LSM Tree
      LSM = write optimized
```

---

### 🏁 The Big Picture

```mermaid
flowchart TD
    Query["📝 SQL Query: SELECT * FROM user WHERE userId = 101"]

    Query --> HasIndex{{"Index exists??"}}

    HasIndex -->|"✅ YES"| BPT["🌲 B+ Tree Lookup\nO(log N)"]
    HasIndex -->|"❌ NO"| FTS["😱 Full Table Scan\nO(N)"]

    BPT --> Clustered{{"Clustered?"}}

    Clustered -->|"✅ YES"| DirectData["🎯 Leaf node = Row data\nReturn directly!"]
    Clustered -->|"❌ NO"| Pointer["🔗 Leaf node = Page pointer\nFetch actual page"]

    Pointer --> ActualPage["📄 Read Data Page\nReturn row"]

    DirectData --> Result["✅ Fast Result!"]
    ActualPage --> Result
    FTS --> AllBlocks["💽 Read ALL 1992 data blocks"] --> Result2["⚠️ Slow Result"]

    style BPT fill:#155724,color:#fff
    style FTS fill:#721c24,color:#fff
    style DirectData fill:#155724,color:#fff
    style Result fill:#155724,color:#fff
    style AllBlocks fill:#721c24,color:#fff
    style Result2 fill:#721c24,color:#fff
```

---

## 📚 References

- **HLD Lecture 15** — SQL Architecture, Data Storage, Indexing, B-Trees, B+ Trees
- SQL Server Documentation — Data Page Structure
- Carnegie Mellon DB Course — B+ Tree Internals
- Martin Kleppmann — *Designing Data-Intensive Applications*

---

*Made with 💙 for the HLD System Design Series*
