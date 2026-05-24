# 🔐 Secure Password Storage & Encryption

> **Core idea**: Passwords are the most common form of authentication — storing them wrong is a catastrophic security failure. This note covers *why*, *what*, and *how* to store them correctly.

---

## 📚 Table of Contents

| # | Topic |
|---|-------|
| 1 | [The Danger of Plain Text Storage](#1️⃣-the-danger-of-plain-text-storage) |
| 2 | [Hashing vs Encryption](#2️⃣-hashing-vs-encryption) |
| 3 | [Salting & Peppering](#3️⃣-salting--peppering) |
| 4 | [Implementation in Node.js (bcrypt)](#4️⃣-implementation-in-nodejs-bcrypt) |
| 5 | [Best Practices](#5️⃣-best-practices) |
| 6 | [Common Mistakes to Avoid](#6️⃣-common-mistakes-to-avoid) |

---

## 1️⃣ The Danger of Plain Text Storage

Storing passwords **exactly as the user typed them** = a ticking time bomb.

```
Database leak → Attacker sees EVERY password instantly → No effort needed
```

**Real consequences:**

| Risk | Impact |
|------|--------|
| **Data Breach** | All user passwords exposed in one shot |
| **Credential Stuffing** | Leaked passwords reused on other sites (Gmail, Banks) |
| **Reputation Damage** | Users lose trust permanently |
| **Legal Consequences** | GDPR, HIPAA fines for mishandling user data |

> ⚠️ Many poorly-designed systems STILL do this. Don't be that engineer.

---

## 2️⃣ Hashing vs Encryption

> People confuse these. Here's the **exact difference**:

| | Hashing | Encryption |
|-|---------|------------|
| **Direction** | One-way (irreversible) | Two-way (reversible) |
| **Purpose** | Verify data integrity / passwords | Transmit data securely |
| **Key needed?** | ❌ No | ✅ Yes (encrypt/decrypt key) |
| **Examples** | bcrypt, SHA-256, Argon2 | AES, RSA |
| **Use for passwords?** | ✅ YES — preferred | ❌ NO — don't decrypt passwords |

**Why hashing for passwords?**

Passwords don't need to be *decrypted* — they only need to be *verified*.

```
Registration:  password → hash()  → store hash in DB
Login:         input    → hash()  → compare with stored hash
                                     ✅ match = login success
                                     ❌ no match = wrong password
```

The original password **never** needs to be recovered. Hashing is perfect for this.

---

## 3️⃣ Salting & Peppering

### 🧂 Salting

A **salt** is a **unique, random string** added to each password *before* hashing.

**Problem it solves — Rainbow Table Attacks:**
```
Without salt:
  hash("password123") = 5f4dcc3b5aa765d...  ← same hash for everyone using same password
  Attacker precomputes a giant table of (plaintext → hash) pairs → instant crack

With salt:
  hash("password123" + "xK9#mP2q") = a8f3e12...  ← unique per user
  hash("password123" + "zL7$nQ4r") = 9c2b7f1...  ← completely different
  → Precomputed table is useless
```

**Benefits:**
- ✅ Prevents rainbow table attacks
- ✅ Same passwords produce different hashes (no pattern leakage)
- ✅ Salt stored alongside hash in DB (it's not secret — it just randomizes)

```
DB stores:  { salt: "xK9#mP2q", hash: "a8f3e12..." }
On login:   hash(input + stored_salt) → compare with stored hash
```

### 🌶️ Peppering

A **pepper** is another secret string added to the password before hashing — but **unlike salt, the pepper is NOT stored in the database**.

```
Registration: hash(password + salt + pepper)  →  store only (salt, hash)
              pepper is kept in application config / environment variable

Login:        hash(input + stored_salt + pepper_from_env)  →  compare
```

| | Salt | Pepper |
|-|------|--------|
| **Stored in** | Database (alongside hash) | Application config / env var |
| **Secret?** | ❌ Not secret | ✅ Must be kept secret |
| **Purpose** | Uniqueness per user | Extra layer if DB is stolen |

**Why pepper helps:**
> Even if the attacker steals the entire database (salts + hashes), they STILL can't crack passwords without the pepper. The pepper lives in your app config — a separate attack surface.

---

## 4️⃣ Implementation in Node.js (bcrypt)

### Install

```bash
npm install bcrypt
```

### Step 1 — Hashing a Password (Registration)

```javascript
const bcrypt = require('bcrypt');

async function hashPassword(password) {
  const saltRounds = 10;  // cost factor — higher = slower = more secure
  const hash = await bcrypt.hash(password, saltRounds);
  return hash;
}
```

> **saltRounds (cost factor):** bcrypt automatically generates a salt internally.
> - `10` = ~100ms per hash (good default for most apps)
> - `12` = ~400ms (more secure, use for sensitive apps)
> - Higher = exponentially slower for attackers trying brute force

### Step 2 — Verifying a Password (Login)

```javascript
async function verifyPassword(inputPassword, storedHash) {
  const match = await bcrypt.compare(inputPassword, storedHash);
  return match; // true = correct password, false = wrong password
}
```

### Full Auth Flow

```
REGISTRATION:
User submits password
      ↓
hashPassword(password)     ← bcrypt handles salting internally
      ↓
Store hash in DB           ← NEVER store plain text
      ↓
✅ Account created

LOGIN:
User submits password
      ↓
Fetch stored hash from DB
      ↓
bcrypt.compare(input, storedHash)
      ↓
✅ true  → Login success
❌ false → Wrong password
```

### Why bcrypt over MD5 / SHA-1?

| Algorithm | Safe? | Why |
|-----------|-------|-----|
| **MD5** | ❌ Broken | Extremely fast → brute force in seconds |
| **SHA-1** | ❌ Broken | Fast, collision vulnerabilities |
| **SHA-256** | ⚠️ Not ideal for passwords | Still too fast — GPUs can try billions/sec |
| **bcrypt** | ✅ Designed for passwords | Intentionally slow, adaptive cost factor |
| **scrypt** | ✅ | Memory-hard, harder to parallelize on GPUs |
| **Argon2** | ✅ Best | Winner of Password Hashing Competition — most modern |

> 🏆 **Recommendation order**: Argon2 > scrypt > bcrypt. bcrypt is fine for most apps.

---

## 5️⃣ Best Practices

```
✅ DO:
  - Always hash passwords with bcrypt / scrypt / Argon2
  - Use a unique salt per user (bcrypt does this automatically)
  - Store pepper in environment variables (not in code or DB)
  - Enforce strong password requirements at registration
  - Implement rate limiting on login endpoints
  - Implement account lockouts after N failed attempts
  - Use HTTPS — never send passwords over plain HTTP
  - Keep libraries up to date (CVE patches)
  - Use a secrets manager (AWS Secrets Manager, Vault) for pepper

❌ DON'T:
  - Never store plain text passwords
  - Never use MD5 or SHA-1 for passwords
  - Never log passwords or hashes anywhere
  - Never reuse the same salt across users
  - Never roll your own crypto — use proven libraries
```

---

## 6️⃣ Common Mistakes to Avoid

| Mistake | Why it's bad | Fix |
|---------|-------------|-----|
| Plain text storage | One DB leak = all accounts gone | Use bcrypt hash |
| MD5 / SHA-1 | Cracked in seconds on modern hardware | Switch to bcrypt/Argon2 |
| Shared salt across users | Same password → same hash → pattern visible | Unique salt per user |
| Logging passwords | Logs often less protected than DB | Never log passwords |
| No password validation | Users use "123" as password | Enforce min length, complexity |
| No brute-force protection | Bot tries millions of combos | Rate limit + lockout |

---

## 🧠 Quick Recap Diagram

```
User Password: "MySecretPass"
      ↓
Add Salt: "xK9#mP2q"  (random, unique per user, stored in DB)
      ↓
Add Pepper: "GlobalSecret" (stored in env var, NOT in DB)
      ↓
bcrypt.hash("MySecretPass" + salt + pepper, 10)
      ↓
Stored in DB: { salt: "xK9#mP2q", hash: "$2b$10$abc123..." }
```

**On Login:**
```
Input: "MySecretPass"
      ↓
Fetch: { salt, hash } from DB
      ↓
bcrypt.compare(input + pepper, hash)
      ↓
✅ Match → Authenticated
❌ No match → Rejected
```

---

> 📖 **Source**: [Secure Password Storage & Encryption — Riddhi Shirohiya (Medium)](https://medium.com/)
