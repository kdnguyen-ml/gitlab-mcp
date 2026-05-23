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
| **Pros** | Zero friction, works everywhere including Docker | Short-lived tokens, auto-refresh, easy revocation | Proper per-user identity for remote deployments | No secrets on server, each user scoped to their own permissions |
| **Cons** | Token never rotates, if leaked it's fully compromised until manually revoked | Needs a browser on the same machine — breaks in Docker/remote | Server must be publicly reachable with HTTPS; most complex setup | Every caller must supply their own token; cannot use SSE transport |
| **Watch out for** | Don't commit the token to git | Non-confidential app uses PKCE (no secret needed); confidential app requires secret | With default passthrough, every MCP client callback URL must be registered in GitLab — use `GITLAB_OAUTH_CALLBACK_PROXY=true` to register only the server URL instead | Session expires after `SESSION_TIMEOUT_SECONDS` (default 1h) |
