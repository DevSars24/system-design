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

```
Client ──▶ API Request + Access Token ──▶ Server
                                              ↓
                                     Verifies access token
                                              ↓
                              ✅ Valid → Return data
                              ❌ Expired → 401 → Client uses refresh token
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

```
Access Token:  15 minutes  → small theft window
Refresh Token: 7 days      → seamless UX (auto-renew without user noticing)

Best of both worlds:
  Security of short-lived access ✅
  UX of long-lived sessions ✅
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
```
localStorage / sessionStorage:
  → Accessible via JS → XSS attack can steal tokens
  → ❌ Not safe for sensitive tokens

HTTP-only Cookie:
  → JS cannot access it (document.cookie blocked)
  → Browser sends it automatically with every request
  → ✅ XSS-proof
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

| ✅ Good for | ❌ Watch out |
|------------|-------------|
| Mobile apps | Must store token somewhere in frontend |
| SPA + REST APIs | In-memory = lost on page refresh |
| Cross-origin APIs | localStorage = XSS risk |

---

### ✅ Method 2: Cookies (Automatic — More Secure)

```
Browser automatically sends:
Cookie: accessToken=xxxx; refreshToken=yyyy
```

No manual attachment needed — browser handles it.

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

| ✅ Good for | ❌ Watch out |
|------------|-------------|
| Web apps (same domain) | Requires CSRF protection (use `sameSite: "Strict"`) |
| More secure against XSS | Cross-origin cookies need `sameSite: "None" + Secure` |
| Seamless UX | Slightly more backend setup |

### Side-by-Side Comparison

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

### Full Sequence

```
1. User makes API request with expired access token

2. Server responds:
   HTTP 401 Unauthorized
   { "error": "Token Expired" }

3. Frontend intercepts the 401 (axios interceptor / fetch wrapper)

4. Frontend sends refresh token to special endpoint:
   POST /api/refresh-token
   Cookie: refreshToken=yyyy...  (auto-sent if using cookies)

5. Backend verifies the refresh token:
   const decoded = jwt.verify(refreshToken, process.env.REFRESH_SECRET);

6. Backend issues a brand new access token:
   const newAccessToken = jwt.sign(
     { userId: decoded.userId },
     process.env.ACCESS_SECRET,
     { expiresIn: "15m" }
   );

7. New access token sent back to client

8. Client retries the original request — user never noticed anything
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

Every time a refresh token is used → **issue a new refresh token too** (and invalidate the old one). This limits damage if a refresh token is stolen.

```
Old Refresh Token → Used → Server issues:
  New Access Token (15m)
  New Refresh Token (7d)  ← old one is now invalidated in DB
```

---

## 6️⃣ Security Deep Dive

### Token Storage Decision Tree

```
Building a web app (same domain)?
  → HTTP-only Cookies
  → Add CSRF protection (sameSite: Strict or CSRF token)

Building a mobile app?
  → Authorization Header
  → Store access token in memory only
  → Store refresh token in secure storage (Keychain / Keystore)

SPA + external API (cross-origin)?
  → Authorization Header
  → Store access token in memory (NOT localStorage)
  → Use refresh token rotation
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

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjMiLCJpYXQiOjE2MjM...
└──────────────────────────────────────┘└──────────────────────────────────────┘
              Header (base64)                     Payload (base64)
              { alg: "HS256", typ: "JWT" }        { userId: "123", exp: ... }

.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
└──────────────────────────────────────────────┘
              Signature (HMAC-SHA256)
              = HMAC(header + "." + payload, SECRET_KEY)
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

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     LOGIN FLOW                               │
│                                                             │
│  User ──▶ POST /login ──▶ bcrypt.compare() ──▶ ✅          │
│                                    │                        │
│                          jwt.sign(access, 15m)              │
│                          jwt.sign(refresh, 7d)              │
│                                    │                        │
│                          Set httpOnly cookies               │
│                                    │                        │
│                          User is logged in ✅               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  API REQUEST FLOW                            │
│                                                             │
│  Request ──▶ Middleware verifyToken()                       │
│                    │                                        │
│           ┌────────┴────────┐                               │
│        ✅ Valid          ❌ Expired                         │
│           │                 │                               │
│       Proceed          POST /refresh-token                  │
│                             │                               │
│                    Verify refresh token                     │
│                             │                               │
│                  ┌──────────┴──────────┐                    │
│               ✅ Valid            ❌ Invalid                 │
│                  │                    │                     │
│          Issue new access token   Force re-login            │
│                  │                                          │
│          Retry original request                             │
└─────────────────────────────────────────────────────────────┘
```

---

> 📖 **Source**: [JWT Authentication — Access Tokens & Refresh Tokens — Ashish Kumar Gupta (Medium)](https://medium.com/)
