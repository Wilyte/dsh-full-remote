# External authentication with Nginx + Authelia

> [中文](./external-auth.zh.md)

This guide walks through the complete configuration for delegating
authentication to an existing Nginx + Authelia (or similar SSO) edge, so
`dsh-full-remote` no longer maintains its own access-token / device-session
/ invite system. A compact deployment reference lives in
[nginx-authelia.md](./nginx-authelia.md); this page is the full
configuration guide — every option explained, plus verification steps.

## 1. When to use this mode

Use `disableTokenAuth: true` when **all** of these hold:

- Your organization already runs a unified authentication gateway
  (typically Nginx with the Authelia `auth_request` flow, or another SSO);
- Every request that reaches the NAS already passed that gateway — either
  because the gateway itself is the public entry, or because your frp /
  tunnel only accepts authenticated traffic;
- You do not want to maintain a second credential system (token rotation,
  device sessions, invites) just for Harness.

When enabled, the proxy keeps doing what it exists for — `Host`/`Origin`
rewriting, header sanitization, CIDR allowlisting, gzip, hashed-asset
caching, the health endpoint, and loopback-only control routes — but it
**stops checking credentials entirely**. The gateway is the only gate.

## 2. Architecture

```
Remote browser ──> Nginx (auth_request ──> Authelia)
                     │  authenticated?
                     ▼
                  frp server (public host)
                     │  frp tunnel
                     ▼
                  NAS: dsh-full-remote  (disableTokenAuth: true)
                     │  Host/Origin rewrite + header sanitization
                     ▼
                  DSH Web: 127.0.0.1:3080
```

Every hop between the browser and the NAS must be covered by the gateway's
authentication — see [Security notes](#8-security-notes) for the failure
modes if it is not.

## 3. Nginx configuration (public server)

A complete `server` block. The two `location` blocks are the heart of the
Authelia integration; the rest is standard reverse-proxy plumbing.

```nginx
server {
    listen 443 ssl;
    server_name dsh.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # ── Authelia verification endpoint ────────────────────────────────
    # Nginx subrequest target. `internal` blocks direct browser access.
    location /authelia/ {
        internal;
        proxy_pass http://127.0.0.1:9091/api/verify;
        proxy_set_header Content-Length "";                      # GET-like subrequest
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;   # where the user was headed
        proxy_set_header X-Original-Method $request_method;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # ── Everything else ───────────────────────────────────────────────
    location / {
        auth_request /authelia/;
        # Unauthenticated → Authelia login; ?rd= returns the user to the
        # originally requested page after login. Keep $host (not $http_host)
        # to avoid Host-header injection.
        error_page 401 =302 https://auth.example.com/?rd=$scheme://$host$request_uri;

        # frp server HTTP vhost port (vhostHTTPPort, default 8080); for a
        # plain TCP tunnel, use that tunnel's remotePort instead.
        proxy_pass http://127.0.0.1:8080;

        proxy_http_version 1.1;
        # WebSocket support — required for live Harness sessions:
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Notes on the Authelia specifics:

- **`auth_request /authelia/`** makes Nginx issue a subrequest to the
  `location /authelia/` block on every request. Authelia answers
  `200` (session valid) or `401` (redirect to login).
- **`X-Original-URL`** must keep the form `$scheme://$http_host$request_uri`
  — Authelia uses it to decide where the user was headed.
- **`error_page 401 =302 .../?rd=...`** — the `?rd=` parameter is the
  post-login return address. Without it, users land on the Authelia portal
  instead of the page they requested. `$host` (not `$http_host`) prevents a
  malicious `Host` header from steering the redirect.
- **WebSocket**: `Upgrade` / `Connection` passthrough is mandatory, or the
  Harness live session breaks.

## 4. frp tunnel configuration

| Question | Answer |
|---|---|
| What does Nginx `proxy_pass` point at? | The frp **server**'s HTTP vhost port — `vhostHTTPPort`, default `8080`. If you run a plain TCP tunnel instead, point it at that tunnel's `remotePort`. |
| What does the frp client forward to? | The NAS address where `dsh-full-remote` listens — keep `listenHost: 127.0.0.1`, `listenPort: 3081`. |
| Does frp forward headers? | Only if configured to. `X-Forwarded-For` must survive the hop unchanged for real-client-IP features (see below). |

Critical: `X-Forwarded-For` passthrough. For the plugin to see the real
client IP, **all three** must hold:

1. Nginx sends `X-Forwarded-For $proxy_add_x_forwarded_for` (shown above);
2. frp forwards that header unchanged to the NAS;
3. `dsh-full-remote` sets `trustForwardedFor: true`.

Otherwise every client collapses to `127.0.0.1` at the proxy — which the
CIDR allowlist **always allows** — so `allowedCidrs` silently stops working
and audit/rate-limit logs show one fake IP.

## 5. dsh-full-remote configuration

Install the plugin, then set `disableTokenAuth: true` in the profile's
patch layer. For the web profile:

```yaml
# /Users/<you>/.dsh/profiles/web/cordis.patch.yml
- id: reverse-proxy
  name: dsh-full-remote
  config:
    disableTokenAuth: true
```

### Option-by-option reference

| Option | Default | In external-auth mode |
|---|---|---|
| `disableTokenAuth` | `false` | **Set `true`.** Skips the token / session / invite gate entirely. |
| `listenHost` | `127.0.0.1` | **Must stay `127.0.0.1`** (loopback) — see Security notes. |
| `listenPort` | `3081` | Keep, or change to whatever frp forwards to. |
| `backendHost` | `127.0.0.1` | Keep loopback — already enforced by validation. |
| `trustForwardedFor` | `false` | Set `true` only if Nginx keeps `X-Forwarded-For` AND frp forwards it (three conditions above). |
| `trustCloudflareConnectingIp` | `false` | Only for a local Cloudflare connector, and only together with `trustForwardedFor: true`. |
| `allowedCidrs` | `[]` | Optional depth-in-depth. Works for real client IPs only when the three XFF conditions hold. |
| `logRequests` | `false` | Useful to confirm the bypass: logs every proxied request. |
| `auditLog` | `true` | Keep on — requests are still audited (no login events, but access/deny events remain). |
| `compressResponses` / `cacheHashedAssets` | `true` | Unaffected by this mode. |

Everything else (timeouts, TLS, `approvalMode`, session settings, invite
settings) is inert while `disableTokenAuth: true` — the gate that used them
is skipped.

## 6. Verification

After restarting `dsh web`:

1. **Panel badge** — Settings → Reverse proxy shows the **external auth
   mode** notice when `disableTokenAuth` is on.
2. **No token required** — an anonymous request through the proxy (e.g.
   `curl http://127.0.0.1:3081/`) reaches the DSH backend directly. There
   is no login redirect, no cookie check.
3. **Control routes still blocked** — `curl http://127.0.0.1:3081/dsh-reverse-proxy/status`
   returns `403`.
4. **Health still served** — `curl http://127.0.0.1:3081/_dsh_reverse_proxy/healthz`
   returns `{"ok":true}`.
5. **Header rewrite intact** — with `logRequests: true`, the debug log shows
   requests forwarded to `127.0.0.1:<backendPort>`.
6. **End-to-end** — open the gateway URL through Nginx in an incognito
   window: you are sent to Authelia, and after login you land on the DSH
   page with settings/credentials/directory browsing working (no 403).

## 7. Switching back

To restore the built-in gate, set `disableTokenAuth: false` (or remove the
line) and restart. Existing tokens, sessions, and invites — if any survived
— work as before; nothing in the proxy is altered by toggling this option.

## 8. Security notes

- **`listenHost` is the exposure vector, not `backendHost`.** With
  `disableTokenAuth: true` there is no credential check at the proxy. If
  `listenHost` is a LAN IP (the UI suggests this for phone-on-WiFi), the
  whole LAN can reach the DSH backend unauthenticated — `allowedCidrs`
  defaults to empty, so there is no backstop. Keep `listenHost: 127.0.0.1`
  and reach the proxy only through the authenticated gateway/tunnel.
- **`allowedCidrs` silently fails under a tunnel** unless the three XFF
  conditions hold (section 4). Loopback is always allowed, so a client that
  collapses to `127.0.0.1` bypasses any CIDR list.
- **The gateway is the only gate.** A misconfigured Nginx (auth disabled on
  some path, or an open port) exposes the whole DSH backend. Verify the
  full path from a browser that has no session.
- **TLS** terminates at Nginx; the frp hop and the NAS listener should be
  considered internal. Keep control routes loopback-only; they cannot be
  reached through the public proxy.
