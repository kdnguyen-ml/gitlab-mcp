# My Understanding Notes

## Authentication Modes

| | PAT | Local OAuth2 | MCP OAuth Proxy | Remote Authorization |
|---|---|---|---|---|
| **Best for** | Local dev, simplest | Local dev, better security | Remote server (Claude.ai, etc.) | Shared server, many users |
| **GitLab setup** | Profile → Access Tokens → generate token | Profile → Applications → create OAuth app (redirect: `localhost:8888`) | Admin → Applications → create OAuth app (redirect: your server's public `/callback`) | None on server — each caller uses their own PAT |
| **Server env vars** | `GITLAB_PERSONAL_ACCESS_TOKEN` | `GITLAB_USE_OAUTH=true` + `GITLAB_OAUTH_CLIENT_ID` + optional `GITLAB_OAUTH_CLIENT_SECRET` | `GITLAB_MCP_OAUTH=true` + `GITLAB_OAUTH_CLIENT_ID` + `GITLAB_OAUTH_CLIENT_SECRET` + `MCP_SERVER_URL` | `REMOTE_AUTHORIZATION=true` + `STREAMABLE_HTTP=true` |
| **Caller provides** | Nothing extra | Nothing (browser popup on first run) | Nothing (browser popup via MCP protocol) | `Authorization: Bearer <token>` header per request |
| **Token lives** | Your env var (static) | `~/.gitlab-mcp-token.json` (auto-refreshed) | Per client session (in-memory) | Caller's own storage |
| **Token rotation** | Manual only | Automatic refresh | Automatic per session | Caller manages their own |
| **Multi-user** | No — one identity for all | No — one browser/token | Yes | Yes — each caller is themselves |
| **Transport** | Any | Any | Streamable HTTP (remote clients) | Streamable HTTP only (no SSE) |
| **HA / Stateless** | ✅ Fully stateless — token is just an env var; share it across pods via Kubernetes Secret | ❌ Breaks in HA — token file lives on one pod's disk; other pods have no token; also requires a local browser | ✅ With stateless mode enabled — session state is encoded cryptographically into the token itself (HMAC + AES-256-GCM), so any pod can serve any request | ✅ Naturally stateless — callers supply their own token per-request; server holds nothing |
| **Pros** | Zero friction, works everywhere including Docker | Short-lived tokens, auto-refresh, easy revocation | Proper per-user identity for remote deployments | No secrets on server, each user scoped to their own permissions |
| **Cons** | Token never rotates, if leaked it's fully compromised until manually revoked | Needs a browser on the same machine — breaks in Docker/remote | Server must be publicly reachable with HTTPS; most complex setup | Every caller must supply their own token; cannot use SSE transport |
| **Watch out for** | Don't commit the token to git | Non-confidential app uses PKCE (no secret needed); confidential app requires secret | With default passthrough, every MCP client callback URL must be registered in GitLab — use `GITLAB_OAUTH_CALLBACK_PROXY=true` to register only the server URL instead. Without stateless mode, OAuth sessions live in one pod's memory — a pod restart or HPA scale-in loses all active sessions | Session expires after `SESSION_TIMEOUT_SECONDS` (default 1h) |

---

## High-Availability Deployment

> Relevant when running multiple server replicas (Kubernetes HPA, load balancer, etc.)

### Which modes work in HA?

| Mode | HA-ready? | Why |
|---|---|---|
| PAT | ✅ Yes | Token is a static env var — all pods share it via a Kubernetes Secret |
| Local OAuth2 | ❌ No | Token stored in a file on one pod's disk; other pods don't have it; also needs a browser |
| MCP OAuth Proxy | ✅ Yes (with stateless mode) | By default sessions are in-memory per pod — enable stateless mode to fix this |
| Remote Authorization | ✅ Yes | Each request carries its own token — server is inherently stateless |

### The problem: OAuth session state lives in memory

Without stateless mode, the MCP OAuth Proxy stores per-user session data (pending auth codes, stored tokens, client/session IDs) in the Node.js process memory of whichever pod handled the initial OAuth flow. If a later request lands on a different pod, that pod has no knowledge of the session — authentication breaks.

### The solution: stateless mode (cryptographic token encoding)

The `stateless/` module solves this by encoding all session state **into the token itself** instead of keeping it in memory:

```
Normal mode (broken under HA):
  Pod A: handles OAuth flow → stores session in memory
  Pod B: receives next request → "Who is this? I have no session." → fails

Stateless mode (HA-safe):
  Pod A: handles OAuth flow → encodes session into a signed/encrypted token
  Pod B: receives next request → decodes the token → has full session context → succeeds
```

**How the encoding works:**

1. **HMAC signing** — proves the token was issued by this server (any pod with the same secret key can verify)
2. **AES-256-GCM encryption** — hides the session contents from the caller; only the server can read it

The `stateless/` module has a layered architecture:

| Layer | What it does |
|---|---|
| `secret` | Derives the signing/encryption key from an env var |
| `codec` | Core HMAC-sign + AES-256-GCM-encrypt / decrypt-verify logic |
| `client-id`, `pending-auth`, `session-id`, `stored-tokens` | Domain-specific encoders for each type of session data |
| `index` | Barrel export — everything else imports from here |

**Required env vars to enable stateless mode:**

```
STREAMABLE_HTTP=true         # required — SSE transport doesn't support stateless mode
# plus the secret key env var for signing/encryption (check docs/environment-variables.md)
```

> `REMOTE_AUTHORIZATION=true` already requires `STREAMABLE_HTTP=true`, so Remote Auth is always HA-ready without extra configuration.

---

## OAuth Token Auto-Refresh (`auth-retry.ts`)

> Applies to **Local OAuth2** and **MCP OAuth Proxy** only. PAT and Remote Auth skip this entirely.

### Background: why tokens expire

OAuth access tokens are deliberately short-lived (typically 2 hours). When one expires, GitLab rejects the next request with **HTTP 401 Unauthorized**. Without auto-refresh you'd have to manually re-authenticate every couple of hours.

### What happens on a 401

Instead of surfacing the error to the caller, the wrapper intercepts it:

```
Your tool call
     │
     ▼
[ fetch wrapper ]──► GitLab API
                         │
                    401 Unauthorized ◄── token expired
                         │
                    [ auto-refresh ]
                    1. Get a new token from GitLab
                    2. Swap the Authorization header
                    3. Replay the original request
                         │
                    200 OK ──────────────────────────► Your tool call gets the result
                              (401 never surfaced)
```

The caller receives the successful response as if nothing happened — the failure is fully transparent.

### The mutex problem: what if 10 requests expire at the same time?

When the token expires, many requests may be in-flight simultaneously. Without coordination:

| Without mutex (broken) | With mutex (correct) |
|---|---|
| Request A hits 401 → starts refresh | Request A hits 401 → starts refresh, sets a shared lock |
| Request B hits 401 → also starts refresh | Request B hits 401 → sees the lock, **waits** on A's refresh |
| Request C hits 401 → also starts refresh | Request C hits 401 → also waits |
| Three parallel refresh calls to GitLab | Only **one** refresh call |
| Refresh token is single-use → B and C's calls fail | All three get the same new token when A's refresh resolves |

The shared lock (`refreshLock`) is called a **mutex** — it coalesces N concurrent refresh attempts into one. All waiters receive the result of that single refresh.

> **HA note:** `refreshLock` is in-process memory — each pod has its own independent mutex. Under HA, two pods may each refresh concurrently, and that is fine. With stateless mode, each pod decodes the session token independently and refreshes as needed without cross-pod coordination. There is no distributed lock to worry about.

### The one case it cannot retry: non-replayable bodies

To replay a request, the wrapper needs the original request body. Two body types are consumed on first read and cannot be re-read:

| Body type | Example | Can replay? |
|---|---|---|
| Plain string / JSON | Most GitLab API calls | ✅ Yes |
| `ReadableStream` | Streaming file upload | ❌ No — already consumed |
| `FormData` | Multipart file upload | ❌ No — already consumed |

When the body is non-replayable, the wrapper gives up and returns the 401 to the caller. File upload endpoints are the main case where this matters.
