# 🔑 JWT Authentication — Access Tokens & Refresh Tokens

> **Core idea**: Modern auth uses two tokens working together — a short-lived access token for API access, and a long-lived refresh token to silently renew it. Understanding *why* (not just *what*) is what separates good engineers from great ones.

---

## 📚 Table of Contents

| # | Topic |
|---|-------|
| 1 | [The Two-Token System](#1️⃣-the-two-token-system) |
| 2 | [Why Short-Lived Access Tokens?](#2️⃣-why-short-lived-access-tokens) |
| 3 | [Complete Backend Flow (Node.js)](#3️⃣-complete-backend-flow-nodejs) |
| 4 | [How Tokens Are Sent](#4️⃣-how-tokens-are-sent) |
| 5 | [Token Refresh Flow](#5️⃣-token-refresh-flow) |
| 6 | [Security Deep Dive](#6️⃣-security-deep-dive) |
| 7 | [Quick Reference](#7️⃣-quick-reference) |

---

## 1️⃣ The Two-Token System

Modern authentication systems use **two tokens** that work in tandem:

| Token | Lifespan | Purpose |
|-------|----------|---------|
| **Access Token** | Short (5–15 minutes) | Authenticate API requests |
| **Refresh Token** | Long (7–30 days) | Generate new access tokens silently |

**Mental model:**

```
Access Token  =  Temporary Entry Pass     (expires fast, limited damage if stolen)
Refresh Token =  Long-Term Identity Card  (used only to renew the entry pass)
```

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: API Request + Access Token
    S->>S: Verify Access Token
    alt Valid Token
        S-->>C: ✅ Return Data
    else Expired Token
        S-->>C: ❌ 401 Unauthorized
        C->>S: POST /api/refresh-token (Refresh Token)
        S-->>C: ✅ New Access Token
        C->>S: Retry Original Request
        S-->>C: ✅ Return Data
    end
```

---

## 2️⃣ Why Short-Lived Access Tokens?

> This is the *why* most developers skip. Don't.

### ❌ Condition 1: Long-Lived Access Token (Bad)

```
Access Token valid for: 30 days

✅ Pro: User never gets logged out (great UX)
❌ Con: Token stolen → attacker has 30-day window of full access
❌ No way to invalidate without a blocklist (JWT is stateless)
→ DANGEROUS
```

### ✅ Condition 2: Short-Lived Access Token (Good)

```
Access Token valid for: 15 minutes

✅ Pro: Stolen token → attacker has 15-minute window max
✅ Pro: System automatically limits unauthorized access window
❌ Con: User must re-login every 15 minutes... (UX terrible)
```

### ✅✅ Solution: Short Access Token + Long Refresh Token

```mermaid
graph LR
    A["Access Token\n(15 min)"] -->|Small theft window| B["✅ Security"]
    C["Refresh Token\n(7 days)"] -->|Auto-renew silently| D["✅ Great UX"]
    B --> E["Best of Both Worlds"]
    D --> E
```

---

## 3️⃣ Complete Backend Flow (Node.js)

### 🟢 Step 1 — User Login Request

```http
POST /api/login
{
  "email": "user@example.com",
  "password": "MySecretPass"
}
```

Backend:
1. Fetch user from DB by email
2. Verify password using `bcrypt.compare(input, storedHash)`
3. If valid → generate both tokens

```mermaid
flowchart TD
    A([User submits login form]) --> B[POST /api/login]
    B --> C{bcrypt.compare\ninput vs storedHash}
    C -- ❌ Wrong Password --> D[401 Unauthorized]
    C -- ✅ Correct --> E[Generate Access Token\nexp: 15m]
    E --> F[Generate Refresh Token\nexp: 7d]
    F --> G[Set httpOnly Cookies]
    G --> H([200 Login Successful])
```

### 🔐 Step 2 — Creating Tokens

```javascript
import jwt from "jsonwebtoken";

// Access Token — short-lived
const accessToken = jwt.sign(
  { userId: user._id },          // payload (keep small)
  process.env.ACCESS_SECRET,     // secret key (different from refresh secret)
  { expiresIn: "15m" }           // expires in 15 minutes
);

// Refresh Token — long-lived
const refreshToken = jwt.sign(
  { userId: user._id },
  process.env.REFRESH_SECRET,    // DIFFERENT secret from access token
  { expiresIn: "7d" }            // expires in 7 days
);
```

> ⚠️ **Critical**: Use **different secrets** for access and refresh tokens. If you use the same secret, a valid refresh token could be used as an access token.

### 🍪 Step 3 — Sending Tokens via HTTP-Only Cookies

```javascript
res
  .cookie("accessToken", accessToken, {
    httpOnly: true,     // ← JS can't read this (XSS protection)
    secure: true,       // ← HTTPS only
    sameSite: "Strict", // ← CSRF protection
    maxAge: 15 * 60 * 1000  // 15 minutes in ms
  })
  .cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: "Strict",
    maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days in ms
  })
  .json({ message: "Login successful" });
```

**Why HTTP-only cookies?**

```mermaid
graph TD
    A[Token Storage Options] --> B[localStorage / sessionStorage]
    A --> C[HTTP-only Cookie]

    B --> D["❌ JS can read it\ndocument.cookie / localStorage.getItem()"]
    D --> E["❌ XSS attack steals token instantly"]

    C --> F["✅ JS cannot access it\nhttpOnly blocks document.cookie"]
    F --> G["✅ XSS-proof\nBrowser sends it automatically"]
```

---

## 4️⃣ How Tokens Are Sent

There are **two patterns** — each with different trade-offs:

### ✅ Method 1: Authorization Header (Common for SPAs & Mobile)

```http
GET /api/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Frontend stores the token (in memory, **not** localStorage) and attaches it manually.

**Backend middleware:**
```javascript
const verifyToken = (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1]; // "Bearer <token>"
  if (!token) return res.status(401).json({ error: "No token" });

  try {
    const decoded = jwt.verify(token, process.env.ACCESS_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ error: "Token expired or invalid" });
  }
};
```

---

### ✅ Method 2: Cookies (Automatic — More Secure)

Browser automatically sends the cookie on every request — no manual attachment needed.

**Backend reads it:**
```javascript
const verifyToken = (req, res, next) => {
  const token = req.cookies.accessToken;
  if (!token) return res.status(401).json({ error: "No token" });

  try {
    const decoded = jwt.verify(token, process.env.ACCESS_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ error: "Token expired" });
  }
};
```

### Side-by-Side Comparison

```mermaid
quadrantChart
    title Token Delivery Method Comparison
    x-axis Low Security --> High Security
    y-axis Hard to Implement --> Easy to Implement
    quadrant-1 Best for Web Apps
    quadrant-2 Avoid
    quadrant-3 Use Carefully
    quadrant-4 Best for Mobile/APIs
    "HTTP-only Cookie": [0.8, 0.65]
    "localStorage + Header": [0.2, 0.85]
    "Memory + Header": [0.7, 0.45]
    "sessionStorage + Header": [0.3, 0.75]
```

| | Authorization Header | Cookies |
|-|---------------------|---------|
| **XSS risk** | ⚠️ If stored in localStorage | ✅ httpOnly = no JS access |
| **CSRF risk** | ✅ Not automatic → safe | ⚠️ Browser auto-sends → need CSRF token |
| **Mobile apps** | ✅ Best choice | ❌ Awkward |
| **Web apps** | ⚠️ Needs careful storage | ✅ Best choice |
| **Cross-origin** | ✅ Easy via headers | ⚠️ Complex SameSite config |

---

## 5️⃣ Token Refresh Flow

> This is the seamless "auto-renew" mechanism — the user never sees a login prompt.

```mermaid
sequenceDiagram
    participant U as User / Browser
    participant C as Client App
    participant S as Server

    U->>C: Triggers API action
    C->>S: GET /api/profile (expired access token)
    S-->>C: 401 Unauthorized - Token Expired

    Note over C: Intercepts 401 automatically
    C->>S: POST /api/refresh-token (sends refresh token)

    alt Refresh Token Valid
        S->>S: jwt.verify(refreshToken, REFRESH_SECRET)
        S->>S: jwt.sign(new access token, 15m)
        S-->>C: 200 + New Access Token (set in cookie)
        C->>S: Retry GET /api/profile (new access token)
        S-->>C: 200 + Profile Data
        C-->>U: ✅ Data displayed (user noticed nothing)
    else Refresh Token Expired / Invalid
        S-->>C: 403 Forbidden
        C-->>U: 🔒 Redirect to Login Page
    end
```

### Backend Refresh Endpoint

```javascript
app.post("/api/refresh-token", (req, res) => {
  const refreshToken = req.cookies.refreshToken;

  if (!refreshToken) return res.status(401).json({ error: "No refresh token" });

  try {
    const decoded = jwt.verify(refreshToken, process.env.REFRESH_SECRET);

    const newAccessToken = jwt.sign(
      { userId: decoded.userId },
      process.env.ACCESS_SECRET,
      { expiresIn: "15m" }
    );

    res.cookie("accessToken", newAccessToken, {
      httpOnly: true,
      secure: true,
      sameSite: "Strict",
      maxAge: 15 * 60 * 1000
    });

    res.json({ message: "Token refreshed" });
  } catch (err) {
    // Refresh token invalid or expired → force re-login
    return res.status(403).json({ error: "Invalid refresh token. Please log in again." });
  }
});
```

### Refresh Token Rotation (Advanced Security)

Every time a refresh token is used → **issue a new refresh token too** (and invalidate the old one):

```mermaid
flowchart LR
    A[Old Refresh Token] -->|Client sends it| B{Server verifies}
    B --> C[Issues New Access Token\n15 min]
    B --> D[Issues New Refresh Token\n7 days]
    B --> E[Invalidates Old Refresh Token\nin DB]
    C --> F([Client stores new tokens])
    D --> F
    E --> G([Old token rejected if reused\n= Token Theft Detected])
```

---

## 6️⃣ Security Deep Dive

### Token Storage Decision Tree

```mermaid
flowchart TD
    A([What are you building?]) --> B{Web App\nsame domain?}
    B -- Yes --> C[Use HTTP-only Cookies]
    C --> D[Add CSRF Protection\nsameSite: Strict]

    B -- No --> E{Mobile App?}
    E -- Yes --> F[Authorization Header]
    F --> G[Access token in memory\nRefresh in Keychain/Keystore]

    E -- No --> H{SPA + External API\ncross-origin?}
    H -- Yes --> I[Authorization Header]
    I --> J[Access token in memory only\nNOT localStorage]
    J --> K[Use Refresh Token Rotation]
```

### Attack Surface & Mitigations

| Attack | Risk | Mitigation |
|--------|------|------------|
| **XSS** | JS steals token from localStorage | Use httpOnly cookies |
| **CSRF** | Attacker tricks browser to send cookie | `sameSite: Strict` + CSRF tokens |
| **Token theft (network)** | Token intercepted in transit | HTTPS only (`secure: true`) |
| **Refresh token leak** | Attacker refreshes indefinitely | Token rotation + DB-backed invalidation |
| **Brute force decode** | Attacker tries to forge tokens | Strong secret keys (256-bit random) |

### JWT Structure (What's Inside)

```mermaid
graph LR
    A["eyJhbGciOiJIUzI1NiJ9\n(Header - base64)"] --> D["."]
    D --> B["eyJ1c2VySWQiOiIxMjMifQ\n(Payload - base64)"]
    B --> E["."]
    E --> C["SflKxwRJSMeKKF2QT4fw\n(Signature - HMAC-SHA256)"]

    A:::header
    B:::payload
    C:::sig

    classDef header fill:#4f46e5,color:#fff
    classDef payload fill:#0891b2,color:#fff
    classDef sig fill:#059669,color:#fff
```

> ⚠️ **JWT payloads are base64 encoded — NOT encrypted.** Anyone can decode and read them. Never put sensitive data (passwords, credit cards) in JWT payload.

---

## 7️⃣ Quick Reference

### Environment Variables (`.env`)

```bash
ACCESS_SECRET=your-super-secret-256-bit-random-key-here
REFRESH_SECRET=another-different-256-bit-random-key-here
```

### Token Lifetimes Cheat Sheet

| Scenario | Access Token | Refresh Token |
|----------|-------------|---------------|
| **High security** (banking) | 5 min | 1 day |
| **Standard app** | 15 min | 7 days |
| **Low security / convenience** | 1 hour | 30 days |

### Complete Auth System — Architecture Overview

```mermaid
flowchart TD
    subgraph LOGIN ["🟢 Login Flow"]
        L1([User submits credentials]) --> L2[POST /login]
        L2 --> L3{bcrypt.compare}
        L3 -- ✅ --> L4[jwt.sign - Access 15m]
        L3 -- ❌ --> L5[401 Unauthorized]
        L4 --> L6[jwt.sign - Refresh 7d]
        L6 --> L7[Set httpOnly Cookies]
        L7 --> L8([Logged In ✅])
    end

    subgraph API ["📡 API Request Flow"]
        A1([API Request]) --> A2[verifyToken middleware]
        A2 --> A3{Token valid?}
        A3 -- ✅ Valid --> A4([Proceed to route handler])
        A3 -- ❌ Expired --> A5[Return 401]
    end

    subgraph REFRESH ["🔁 Refresh Flow"]
        R1([Client catches 401]) --> R2[POST /refresh-token]
        R2 --> R3{Refresh token valid?}
        R3 -- ✅ --> R4[Issue new Access Token]
        R4 --> R5[Retry original request]
        R3 -- ❌ --> R6([Force re-login 🔒])
    end

    L8 --> A1
    A5 --> R1
```

---

> 📖 **Source**: [JWT Authentication — Access Tokens & Refresh Tokens — Ashish Kumar Gupta (Medium)](https://medium.com/)
