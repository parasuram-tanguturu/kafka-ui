# CORS and Development Proxy

> Understanding why `VITE_DEV_PROXY` is needed and how it works.

---

## The Problem: Same-Origin Policy

Browsers enforce a security rule called **Same-Origin Policy** to protect users from malicious websites.

### Why It Exists

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     THE THREAT MODEL                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   You're logged into your bank at bank.com (session cookie stored)         │
│                                                                             │
│   You visit evil.com which has JavaScript:                                  │
│                                                                             │
│   fetch('https://bank.com/api/transfer', {                                  │
│     method: 'POST',                                                         │
│     body: { to: 'hacker', amount: 10000 }                                  │
│   });                                                                       │
│                                                                             │
│   WITHOUT Same-Origin Policy:                                               │
│   ❌ evil.com steals your money using YOUR browser's cookies!               │
│                                                                             │
│   WITH Same-Origin Policy:                                                  │
│   ✅ Browser blocks it: "evil.com cannot access bank.com"                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What is an "Origin"?

An origin is defined by: **Protocol + Host + Port**

```
   http://localhost:3000/app
   ─┬──   ─────┬───── ─┬──
    │         │       │
    │         │       └── Path (doesn't matter for origin)
    │         │
    │         └────────── Host + Port  ← THESE DEFINE ORIGIN
    │
    └──────────────────── Protocol
```

**Examples:**

| URL A | URL B | Same Origin? |
|-------|-------|--------------|
| `http://localhost:3000` | `http://localhost:3000/api` | ✅ Yes |
| `http://localhost:3000` | `http://localhost:8080` | ❌ No (different port) |
| `http://example.com` | `https://example.com` | ❌ No (different protocol) |
| `http://api.example.com` | `http://example.com` | ❌ No (different host) |

---

## CORS: Cross-Origin Resource Sharing

CORS is a mechanism that allows servers to opt-in to cross-origin requests.

### How CORS Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORS FLOW                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Browser sends request with Origin header:                              │
│                                                                             │
│      ┌──────────────────────────────────────────────────────────┐          │
│      │ GET /api/clusters HTTP/1.1                               │          │
│      │ Host: localhost:8080                                     │          │
│      │ Origin: http://localhost:3000   ← Browser adds this     │          │
│      └──────────────────────────────────────────────────────────┘          │
│                              │                                              │
│                              ▼                                              │
│   2. Server responds with CORS headers (or not):                            │
│                                                                             │
│      ALLOWED:                                                               │
│      ┌──────────────────────────────────────────────────────────┐          │
│      │ HTTP/1.1 200 OK                                          │          │
│      │ Access-Control-Allow-Origin: http://localhost:3000       │          │
│      │ ...response body...                                      │          │
│      └──────────────────────────────────────────────────────────┘          │
│      ✅ Browser allows JavaScript to read the response                     │
│                                                                             │
│      BLOCKED:                                                               │
│      ┌──────────────────────────────────────────────────────────┐          │
│      │ HTTP/1.1 200 OK                                          │          │
│      │ (no Access-Control-Allow-Origin header)                  │          │
│      │ ...response body...                                      │          │
│      └──────────────────────────────────────────────────────────┘          │
│      ❌ Browser blocks JavaScript from reading the response                │
│         (Note: request WAS sent, response WAS received,                    │
│          but JS cannot ACCESS it)                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Preflight Requests (for "Complex" Requests)

For requests with custom headers, methods like PUT/DELETE, or JSON content-type:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PREFLIGHT FLOW                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Browser: "Before I send the real request, let me check..."            │
│                                                                             │
│      ┌──────────────────────────────────────────────────────────┐          │
│      │ OPTIONS /api/users HTTP/1.1                              │          │
│      │ Origin: http://localhost:3000                            │          │
│      │ Access-Control-Request-Method: POST                      │          │
│      │ Access-Control-Request-Headers: Content-Type             │          │
│      └──────────────────────────────────────────────────────────┘          │
│                              │                                              │
│                              ▼                                              │
│   2. Server: "Yes, I allow that"                                           │
│                                                                             │
│      ┌──────────────────────────────────────────────────────────┐          │
│      │ HTTP/1.1 204 No Content                                  │          │
│      │ Access-Control-Allow-Origin: http://localhost:3000       │          │
│      │ Access-Control-Allow-Methods: GET, POST, PUT, DELETE     │          │
│      │ Access-Control-Allow-Headers: Content-Type               │          │
│      └──────────────────────────────────────────────────────────┘          │
│                              │                                              │
│                              ▼                                              │
│   3. Browser: "OK, now I'll send the real request"                         │
│                                                                             │
│      ┌──────────────────────────────────────────────────────────┐          │
│      │ POST /api/users HTTP/1.1                                 │          │
│      │ Origin: http://localhost:3000                            │          │
│      │ Content-Type: application/json                           │          │
│      │ {"name": "John"}                                         │          │
│      └──────────────────────────────────────────────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Development Problem

In local development for Kafbat UI:

| Component | URL | Purpose |
|-----------|-----|---------|
| Frontend (Vite) | `http://localhost:3000` | React app with hot reload |
| Backend (Spring) | `http://localhost:8080` | REST API |

**These are different origins!**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WITHOUT PROXY                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Browser (localhost:3000)                                                  │
│        │                                                                    │
│        │ JavaScript: fetch('http://localhost:8080/api/clusters')           │
│        │                                                                    │
│        ▼                                                                    │
│   Browser Security Engine:                                                  │
│   "Request from :3000 to :8080 → Cross-origin!"                            │
│   "Checking for CORS headers..."                                           │
│        │                                                                    │
│        ▼                                                                    │
│   Spring Boot :8080                                                         │
│   (doesn't send Access-Control-Allow-Origin in local profile)              │
│        │                                                                    │
│        ▼                                                                    │
│   Browser: "No CORS header → BLOCKED!" ❌                                   │
│                                                                             │
│   Console Error:                                                            │
│   "Access to fetch at 'http://localhost:8080/api/clusters' from origin     │
│    'http://localhost:3000' has been blocked by CORS policy"                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Solution: Development Proxy

### Key Insight

**CORS is a browser-only restriction. Servers talking to servers have no CORS restrictions.**

### How the Proxy Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WITH VITE_DEV_PROXY                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Browser (localhost:3000)                                                  │
│        │                                                                    │
│        │ JavaScript: fetch('/api/clusters')                                │
│        │             ───────────────────────                                │
│        │             Relative URL = same origin!                            │
│        ▼                                                                    │
│   Browser Security Engine:                                                  │
│   "Request from :3000 to :3000 → Same origin → ALLOWED" ✅                 │
│        │                                                                    │
│        ▼                                                                    │
│   Vite Dev Server (localhost:3000)                                          │
│   "URL matches /api/* pattern"                                              │
│   "Forwarding to VITE_DEV_PROXY (http://localhost:8080)"                   │
│        │                                                                    │
│        │ HTTP Request (server-to-server, NO CORS!)                         │
│        ▼                                                                    │
│   Spring Boot (localhost:8080)                                              │
│   "Here's the data" (doesn't know/care about browser)                      │
│        │                                                                    │
│        │ Response                                                           │
│        ▼                                                                    │
│   Vite Dev Server                                                           │
│   "Got response, sending back to browser"                                  │
│        │                                                                    │
│        ▼                                                                    │
│   Browser                                                                   │
│   "Response from :3000 (my origin) → ALLOWED" ✅                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Vite Configuration

From `frontend/vite.config.ts`:

```typescript
// Line 11: Read the environment variable
const isProxy = process.env.VITE_DEV_PROXY;

// Lines 90-116: Configure proxy rules
proxy: {
  '/api': {
    target: isProxy,        // http://localhost:8080
    changeOrigin: true,     // Changes Origin header to target
    secure: false,          // Allow self-signed certs
  },
  '/login': { target: isProxy, ... },
  '/logout': { target: isProxy, ... },
  '/actuator/info': { target: isProxy, ... },
}
```

### Proxied Routes

| Frontend Request | Vite Forwards To |
|------------------|------------------|
| `GET /api/clusters` | `http://localhost:8080/api/clusters` |
| `POST /api/topics` | `http://localhost:8080/api/topics` |
| `POST /login` | `http://localhost:8080/login` |
| `POST /logout` | `http://localhost:8080/logout` |
| `GET /actuator/info` | `http://localhost:8080/actuator/info` |

---

## Usage

### Starting Frontend with Proxy

```bash
cd frontend

# Set the proxy target and start dev server
VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

### Alternative: Using .env.local File

Create `frontend/.env.local`:

```bash
VITE_DEV_PROXY=http://localhost:8080
```

Then just run:

```bash
pnpm dev
```

### If Backend is on Different Port

```bash
# Backend on 8081
./gradlew :api:bootRun --args='--spring.profiles.active=local --server.port=8081'

# Frontend proxy to 8081
VITE_DEV_PROXY=http://localhost:8081 pnpm dev
```

---

## Comparison Table

| Aspect | Without Proxy | With Proxy |
|--------|---------------|------------|
| **Frontend fetch URL** | `http://localhost:8080/api/...` | `/api/...` |
| **Browser perceives** | Cross-origin (3000 → 8080) | Same-origin (3000 → 3000) |
| **CORS check** | ❌ Fails (no headers) | ✅ Not needed |
| **Who calls backend** | Browser directly | Vite server |
| **Works without config** | ❌ No | ✅ Yes |

---

## Why This is Safe

**In development:**
- You control both servers
- Both are on localhost
- No malicious third-party involved
- It's YOUR browser, YOUR machine

**In production:**
- Frontend is bundled into the Spring Boot JAR
- Both served from the SAME origin (e.g., `https://kafka-ui.example.com`)
- CORS is not an issue (same origin)

---

## Common Errors

### "CORS policy: No 'Access-Control-Allow-Origin' header"

**Cause:** Forgot to set `VITE_DEV_PROXY`

**Fix:**
```bash
VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

### "Failed to fetch" or "Network Error"

**Cause:** Backend not running

**Fix:**
```bash
# Start backend first
./gradlew :api:bootRun --args='--spring.profiles.active=local'

# Then start frontend
VITE_DEV_PROXY=http://localhost:8080 pnpm dev
```

### "Proxy error: ECONNREFUSED"

**Cause:** Backend crashed or wrong port

**Fix:**
```bash
# Check backend is running
curl http://localhost:8080/api/clusters

# If error, restart backend
./gradlew :api:bootRun --args='--spring.profiles.active=local'
```

---

## Related Documentation

- [MDN: Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Vite: Server Proxy](https://vitejs.dev/config/server-options.html#server-proxy)
