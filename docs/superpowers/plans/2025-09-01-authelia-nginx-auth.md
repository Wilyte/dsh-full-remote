# Authelia + Nginx 外部认证实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development (recommended) or executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 dsh-full-remote 新增 `disableTokenAuth` 配置项，当启用时跳过内置令牌/会话/邀请鉴权，把认证职责完全外包给前置的 Nginx + Authelia，同时保留 Host/Origin 改写、头部剥离、压缩等代理功能。

**Architecture:** 新增一个布尔配置 `disableTokenAuth`（默认 `false`，向后兼容）。当为 `true` 时，代理的 `onRequest` 和 `upgrade` 处理器跳过登录门、等待页、设备页、会话校验四个环节，直接将请求转发到 DSH Web 后端。控制路由（`/dsh-reverse-proxy/*`）、CIDR 白名单、healthz 保留。设置面板显示认证模式徽标。

**Tech Stack:** Node.js http/https、TypeScript、Schemastery（配置 schema）、node:test + vitest（测试）、React（客户端面板）。

**Spec:** [docs/superpowers/specs/2025-09-01-authelia-nginx-auth-design.md](../specs/2025-09-01-authelia-nginx-auth-design.md)

## Global Constraints

- `disableTokenAuth` 默认值为 `false`，现有部署零感知。
- 控制路由 `CONTROL_PREFIX = '/dsh-reverse-proxy'`（来自 `src/control.ts`）在禁用鉴权时仍然必须返回 403，不允许任何远程请求到达。
- 登录门路径常量来自 `src/pages.ts`：`GATE_PREFIX = '/_dsh_reverse_proxy'`，`LOGIN_PATH`、`HOME_PATH`、`HOME_RENAME_PATH`、`LOGOUT_PATH`、`HEALTHZ_PATH`、`WAIT_PREFIX`。
- 禁用鉴权后 `proxyRequest` 的 `sessionId` 参数传固定字符串 `'external-auth'`，连接跟踪/关闭逻辑无需改动。
- 所有用户可见文案走 `src/client/i18n.ts` 的 `zh`/`en` 字典，两字典键必须一致（`tests/remote.client.test.tsx` 中有键一致性断言）。
- 测试命令：`pnpm run test:unit`（node:test，含 `tests/plugin.test.ts`）与 `pnpm run test:client`（vitest，含 `tests/remote.client.test.tsx`）。

---

### Task 1: 配置 schema 新增 `disableTokenAuth`

**Files:**
- Modify: `src/config.ts`

**Interfaces:**
- Produces: `Config` 对象新增 `disableTokenAuth: boolean` 字段，默认 `false`。`RuntimeConfig = ReturnType<typeof Config>` 自动包含该字段。

- [ ] **Step 1: 在 `src/config.ts` 的 Config schema 中新增字段**

在 `cacheHashedAssets` 字段之后（第 55 行）添加：

```ts
  cacheHashedAssets: Schema.boolean().default(true).description('When true, add Cache-Control: public, max-age=31536000, immutable on successful hashed /assets/* responses that have no upstream cache header. index.html and /api are never cached.'),
  disableTokenAuth: Schema.boolean().default(false).description('When true, skip the built-in access-token / device-session / invite gate entirely and forward every request straight to the backend. Intended for deployments where an external reverse proxy (e.g. Nginx + Authelia) already authenticated the client. Control routes stay loopback-only; CIDR allowlist, header sanitization, gzip, caching, and audit logging still apply.'),
```

- [ ] **Step 2: 运行类型检查确认 schema 类型正确**

Run: `pnpm run typecheck`
Expected: PASS（无类型错误，`RuntimeConfig` 自动包含 `disableTokenAuth: boolean`）

- [ ] **Step 3: 提交**

```bash
git add src/config.ts
git commit -m "feat(config): add disableTokenAuth option for external auth bypass"
```

---

### Task 2: 代理层跳过内置鉴权

**Files:**
- Modify: `src/proxy.ts`
- Test: `tests/plugin.test.ts`

**Interfaces:**
- Consumes: `Config.disableTokenAuth: boolean`（Task 1）
- Produces: `ProxySpec.disableTokenAuth?: boolean`；`listenProxy` 在 `disableTokenAuth === true` 时行为变化（见下方测试断言）。
- 内部常量：`const EXTERNAL_AUTH_SESSION_ID = 'external-auth'`（`proxyRequest` 的 `sessionId` 参数）。

- [ ] **Step 1: 在 `ProxySpec` 接口中新增可选字段**

在 `src/proxy.ts` 的 `ProxySpec` 接口（约第 113 行 `cacheHashedAssets?: boolean` 之后）添加：

```ts
  /** When true, skip the built-in token/session/invite gate and forward every
   *  request straight to the backend. Intended for an external reverse proxy
   *  (e.g. Nginx + Authelia) that already authenticated the client. Control
   *  routes stay loopback-only. */
  disableTokenAuth?: boolean
```

- [ ] **Step 2: 在模块顶部定义外部认证会话 ID 常量**

在 `src/proxy.ts` 顶部常量区（`const DEFAULT_SESSION_MAX_AGE_SECONDS` 附近，约第 140 行）添加：

```ts
/** Session id used for every forwarded request when disableTokenAuth bypasses
 *  the built-in login gate. Only feeds connection tracking; never persisted. */
const EXTERNAL_AUTH_SESSION_ID = 'external-auth'
```

- [ ] **Step 3: 修改 `onRequest` 处理器跳过鉴权门**

在 `src/proxy.ts` 的 `listenProxy` 内的 `onRequest`（约第 701-736 行），将健康检查之后的登录门路由改为条件执行，并在会话校验前插入外部认证直通分支。将：

```ts
    if (path === HEALTHZ_PATH) {
      res.writeHead(200, { 'content-type': 'application/json; charset=utf-8', 'cache-control': 'no-store' })
      res.end('{"ok":true}\n')
      return
    }
    if (path === LOGIN_PATH) {
      await handleLogin(req, res, runtimeSpec)
      return
    }
    if (handleWaitRoutes(req, res, runtimeSpec, path, cookie)) return
    if (await handleDevicePages(req, res, runtimeSpec, path, cookie)) return
    if (path.startsWith(spec.controlPrefix)) {
      drainRequest(req)
      sendText(res, 403, 'forbidden\n')
      return
    }
    const session = runtimeSpec.sessionStore.validate(cookie, effectiveRemoteAddress(req, runtimeSpec))
```

替换为：

```ts
    if (path === HEALTHZ_PATH) {
      res.writeHead(200, { 'content-type': 'application/json; charset=utf-8', 'cache-control': 'no-store' })
      res.end('{"ok":true}\n')
      return
    }
    if (path.startsWith(spec.controlPrefix)) {
      drainRequest(req)
      sendText(res, 403, 'forbidden\n')
      return
    }
    if (spec.disableTokenAuth === true) {
      // External auth (e.g. Nginx + Authelia) already authenticated the
      // client: skip login / wait / device pages and session validation,
      // forward everything else straight to the backend.
      proxyRequest(req, res, runtimeSpec, EXTERNAL_AUTH_SESSION_ID)
      return
    }
    if (path === LOGIN_PATH) {
      await handleLogin(req, res, runtimeSpec)
      return
    }
    if (handleWaitRoutes(req, res, runtimeSpec, path, cookie)) return
    if (await handleDevicePages(req, res, runtimeSpec, path, cookie)) return
    const session = runtimeSpec.sessionStore.validate(cookie, effectiveRemoteAddress(req, runtimeSpec))
```

注意：把 `controlPrefix` 检查提前到登录门之前。这不会改变现有行为——`/dsh-reverse-proxy` 与 `/_dsh_reverse_proxy` 前缀互不重叠（前者无下划线），且当前代码中该检查本就在登录门之后、会话校验之前，二者不会同时命中。

- [ ] **Step 4: 修改 `upgrade` 处理器跳过会话校验**

在 `src/proxy.ts` 的 `server.on('upgrade', ...)` 内（约第 751-776 行），修改会话校验条件。将：

```ts
    const path = pathnameOf(req.url)
    const cookie = parseCookies(req.headers.cookie)[spec.cookieName]
    const session = runtimeSpec.sessionStore.validate(cookie, upgradeRemote)
    if (path.startsWith(spec.controlPrefix) || session === undefined) {
```

替换为：

```ts
    const path = pathnameOf(req.url)
    const cookie = parseCookies(req.headers.cookie)[spec.cookieName]
    if (path.startsWith(spec.controlPrefix)) {
      upgradeTracker.fail(upgradeRemote)
      upgradeTracker.prune()
      denySocket(socket, '403 Forbidden')
      spec.audit?.('access.denied', { reason: 'control', remote: upgradeRemote, path: pathnameOf(req.url), upgrade: true })
      return
    }
    if (spec.disableTokenAuth === true) {
      // External auth already applied; forward the upgrade without a session.
      const headers = forwardHeaders(req, rewriteAuthority, {
        tls: tlsEnabled,
        trustForwardedProto: runtimeSpec.trustForwardedProto(),
        forwardedFor: upgradeRemote,
      })
      headers.connection = 'Upgrade'
      headers.upgrade = req.headers.upgrade ?? 'websocket'
      const up = httpRequest({
        hostname: spec.backendHost,
        port: spec.backendPort,
        path: req.url,
        method: 'GET',
        headers,
      })
      trackUpstream(up)
      up.setTimeout(spec.upstreamTimeoutMs, () => { up.destroy() })
      up.on('upgrade', (upRes, upSocket, upHead) => {
        trackUpstream(upSocket)
        upgradeTracker.success(upgradeRemote)
        spec.audit?.('ws.open', { remote: upgradeRemote, path: pathnameOf(req.url), status: upRes.statusCode ?? 101 })
        spec.log?.({ method: req.method, path: pathnameOf(req.url), status: upRes.statusCode ?? 101, remote: upgradeRemote })
        writeRawHead(
          socket,
          upRes.statusCode ?? 101,
          upRes.statusMessage ?? 'Switching Protocols',
          sanitizeUpgradeResponseHeaders(upRes.headers),
        )
        if (upHead.length > 0) socket.write(upHead)
        if (head.length > 0) upSocket.write(head)
        upSocket.pipe(socket)
        socket.pipe(upSocket)
      })
      up.on('response', (upRes) => {
        spec.audit?.('ws.reject', { remote: upgradeRemote, path: pathnameOf(req.url), status: upRes.statusCode ?? 502 })
        upRes.resume()
        writeRawHead(socket, upRes.statusCode ?? 502, upRes.statusMessage ?? 'Bad Gateway', {
          ...sanitizeResponseHeaders(upRes.headers),
          Connection: 'close',
        })
        socket.destroy()
      })
      up.on('error', () => { socket.destroy() })
      socket.on('error', () => { up.destroy() })
      up.end()
      return
    }
    const session = runtimeSpec.sessionStore.validate(cookie, upgradeRemote)
    if (session === undefined) {
```

注意：这段与原有非禁用路径的转发代码几乎一致，差异仅在于没有 `releaseSession` 会话跟踪（外部认证模式下无会话可吊销）。`upgradeRetryAfter` 限速检查位于本替换文本之上、保持不变——外部认证模式下成功升级会 `upgradeTracker.success()` 清桶，因此该检查实际不会锁定正常流量，保留它只是维持防刷语义。控制前缀的升级请求在原代码中与「无会话」共用 401 拒绝分支，本替换将其拆为独立的 403 分支（与控制路由的 HTTP 行为一致）；`upgradeTracker.fail()` 与 `prune()` 仍在两个拒绝分支中各自调用。

- [ ] **Step 5: 编写失败测试（单元测试）**

在 `tests/plugin.test.ts` 末尾追加 `describe('disableTokenAuth external auth bypass', ...)`：

```ts
describe('disableTokenAuth external auth bypass', () => {
  async function bypassProxy(overrides: Record<string, unknown> = {}) {
    const seen: Array<{ host: string | undefined, marker: string | undefined }> = []
    const backend = createServer((req, res) => {
      seen.push({ host: req.headers.host, marker: req.headers['x-dsh-reverse-proxy'] })
      res.writeHead(200, { 'content-type': 'application/json' })
      res.end(JSON.stringify({ ok: true }))
    })
    await new Promise<void>((resolve, reject) => {
      backend.once('error', reject)
      backend.listen(0, '127.0.0.1', resolve)
    })
    cleanups.push(() => new Promise<void>(resolve => backend.close(() => resolve())))
    const proxy = await listenProxy({
      listenHost: '127.0.0.1',
      listenPort: 0,
      backendHost: '127.0.0.1',
      backendPort: portOf(backend),
      accessToken: 'irrelevant-token',
      cookieName: 'session',
      controlPrefix: '/dsh-reverse-proxy',
      maxRequestBytes: 4096,
      upstreamTimeoutMs: 2000,
      disableTokenAuth: true,
      ...overrides,
    })
    cleanups.push(proxy.close)
    return { proxy, seen }
  }

  it('forwards anonymous requests straight to the backend with no cookie', async () => {
    const { proxy, seen } = await bypassProxy()
    const response = await http({ port: proxy.port, path: '/api/example' })
    assert.equal(response.status, 200)
    assert.deepEqual(JSON.parse(response.body), { ok: true })
    assert.equal(seen[0].marker, '1')
  })

  it('skips the login gate: GET /_dsh_reverse_proxy/login reaches the backend', async () => {
    const { proxy, seen } = await bypassProxy()
    const response = await http({ port: proxy.port, path: '/_dsh_reverse_proxy/login' })
    // No login page: the request is proxied, so the backend's JSON arrives.
    assert.equal(response.status, 200)
    assert.deepEqual(JSON.parse(response.body), { ok: true })
    assert.equal(seen.length, 1)
  })

  it('still blocks control routes with 403', async () => {
    const { proxy } = await bypassProxy()
    const response = await http({ port: proxy.port, path: '/dsh-reverse-proxy/status' })
    assert.equal(response.status, 403)
  })

  it('still serves the health endpoint', async () => {
    const { proxy } = await bypassProxy()
    const response = await http({ port: proxy.port, path: '/_dsh_reverse_proxy/healthz' })
    assert.equal(response.status, 200)
    assert.deepEqual(JSON.parse(response.body), { ok: true })
  })

  it('still applies the CIDR allowlist before bypassing auth', async () => {
    const { proxy } = await bypassProxy({ ipAllowed: () => false })
    const response = await http({ port: proxy.port, path: '/api/example' })
    assert.equal(response.status, 403)
  })

  it('accepts WebSocket upgrades without a session cookie', async () => {
    const { proxy } = await bypassProxy()
    const ws = await wsHandshake({ port: proxy.port, path: '/ws' })
    assert.equal(ws.status.startsWith('HTTP/1.1 101'), true)
    ws.socket.destroy()
  })
})
```

注意：`seen[0].host` 断言会因测试后端未记录端口而复杂化，简化为只断言 `seen[0].marker === '1'` 与 Host 头含 `127.0.0.1`：

```ts
  it('still rewrites Host to loopback while bypassing auth', async () => {
    const { proxy, seen } = await bypassProxy()
    await http({ port: proxy.port, path: '/api/example' })
    assert.match(seen[0].host ?? '', /^127\.0\.0\.1:\d+$/)
  })
```

- [ ] **Step 6: 运行测试验证失败**

Run: `pnpm run test:unit -- --test-name-pattern "disableTokenAuth"`（先确认此命令可运行；若 node:test 过滤参数不可用则运行 `node --experimental-strip-types --test tests/plugin.test.ts` 并观察失败）
Expected: FAIL —— `disableTokenAuth` 未被 `ProxySpec` 接受或行为未生效（类型错误 / 匿名请求被重定向到登录页）。

- [ ] **Step 7: 运行全部单测验证通过**

Run: `pnpm run test:unit`
Expected: PASS（全部测试，包括新增的 6 个断言块）

- [ ] **Step 8: 运行 lint 与类型检查**

Run: `pnpm run lint && pnpm run typecheck`
Expected: PASS

- [ ] **Step 9: 提交**

```bash
git add src/proxy.ts tests/plugin.test.ts
git commit -m "feat(proxy): bypass token gate when disableTokenAuth is enabled"
```

---

### Task 3: 运行时接线 + 状态暴露

**Files:**
- Modify: `src/index.ts`
- Modify: `src/runtime-types.ts`

**Interfaces:**
- Consumes: `Config.disableTokenAuth`（Task 1）、`ProxySpec.disableTokenAuth`（Task 2）
- Produces: `RuntimeStatus.disableTokenAuth: boolean`；`snapshot()` 返回该字段。

- [ ] **Step 1: 在 `RuntimeStatus` 接口中新增字段**

在 `src/runtime-types.ts` 的 `RuntimeStatus` 接口中（`trustForwardedFor` 之后）添加：

```ts
  /** True when the built-in token/session gate is bypassed (external auth). */
  disableTokenAuth: boolean
```

- [ ] **Step 2: 在 `snapshot()` 中返回该字段**

在 `src/index.ts` 的 `snapshot()`（约第 219-241 行）返回值对象中，`trustForwardedFor` 之后添加：

```ts
      trustForwardedFor: config.trustForwardedFor === true,
      disableTokenAuth: config.disableTokenAuth === true,
```

- [ ] **Step 3: 启动代理时传入 `disableTokenAuth`**

在 `src/index.ts` 的 `start()` 中调用 `listenProxy({...})`（约第 313-353 行），在 `cacheHashedAssets` 之后添加：

```ts
        cacheHashedAssets: config.cacheHashedAssets !== false,
        disableTokenAuth: config.disableTokenAuth === true,
```

- [ ] **Step 4: 验证类型与现有测试**

Run: `pnpm run typecheck && pnpm run test:unit`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add src/index.ts src/runtime-types.ts
git commit -m "feat(runtime): wire disableTokenAuth through runtime and expose in status"
```

---

### Task 4: 设置面板显示认证模式

**Files:**
- Modify: `src/client/types.ts`
- Modify: `src/client/i18n.ts`
- Modify: `src/client/RemoteSection.tsx`
- Test: `tests/remote.client.test.tsx`

**Interfaces:**
- Consumes: `RuntimeStatus.disableTokenAuth: boolean`（Task 3，经控制面 JSON 传到客户端）
- Produces: `ProxyStatus.disableTokenAuth?: boolean`；新 i18n 键 `status.externalAuth`（zh/en）。

- [ ] **Step 1: 客户端 `ProxyStatus` 类型新增字段**

在 `src/client/types.ts` 的 `ProxyStatus` 中（`trustForwardedFor?: boolean` 之后）添加：

```ts
  /** True when the built-in token/session gate is bypassed (external auth). */
  disableTokenAuth?: boolean
```

- [ ] **Step 2: i18n 字典新增键（zh 与 en 都要加）**

在 `src/client/i18n.ts` 的 `zh` 字典（`'status.loadingHint'` 之后，约第 28 行）添加：

```ts
  'status.externalAuth': '外部认证模式：已跳过内置令牌门，认证由前置网关（如 Nginx + Authelia）负责。',
```

在 `en` 字典对应位置（`Record<keyof typeof zh, string>`）添加：

```ts
  'status.externalAuth': 'External auth mode: built-in token gate is bypassed; authentication is handled by an upstream gateway (e.g. Nginx + Authelia).',
```

- [ ] **Step 3: 状态卡片显示外部认证徽标**

在 `src/client/RemoteSection.tsx` 的 JSX 中，找到状态卡片标题/提示区域（`heroTitle`/`heroHint` 渲染处，约第 430-470 行），在状态提示之后插入条件徽标：

```tsx
      {status?.disableTokenAuth === true && (
        <p className={css.externalAuthBadge} role="status">{t('status.externalAuth')}</p>
      )}
```

在 `src/client/remote.module.css` 中新增样式：

```css
.externalAuthBadge {
  margin-top: 8px;
  padding: 6px 10px;
  border: 1px solid var(--dsh-color-border, #d0d7de);
  border-radius: 6px;
  background: var(--dsh-color-bg-subtle, #f6f8fa);
  color: var(--dsh-color-text-secondary, #57606a);
  font-size: 12px;
  line-height: 1.5;
}
```

- [ ] **Step 4: 编写客户端测试**

在 `tests/remote.client.test.tsx` 中新增测试：

```tsx
describe('external auth mode badge', () => {
  it('shows the external-auth notice when disableTokenAuth is on', async () => {
    const service = api({
      status: vi.fn().mockResolvedValue({ ...stopped, disableTokenAuth: true }),
    })
    render(<RemoteSection {...sectionProps(service)} />)
    expect(await screen.findByText(/外部认证模式/)).toBeTruthy()
  })

  it('does not show the notice when disableTokenAuth is off', () => {
    const service = api()
    render(<RemoteSection {...sectionProps(service)} />)
    expect(screen.queryByText(/外部认证模式/)).toBeNull()
  })
})
```

- [ ] **Step 5: 运行客户端测试**

Run: `pnpm run test:client`
Expected: PASS（含新增 2 个测试与原有 i18n 键一致性断言）

- [ ] **Step 6: 运行完整检查**

Run: `pnpm run check`
Expected: PASS（lint + typecheck + 单测 + 客户端测试 + build）

- [ ] **Step 7: 提交**

```bash
git add src/client/types.ts src/client/i18n.ts src/client/RemoteSection.tsx src/client/remote.module.css tests/remote.client.test.tsx
git commit -m "feat(client): show external-auth mode badge in reverse proxy panel"
```

---

### Task 5: 文档更新

**Files:**
- Modify: `README.md`
- Modify: `README.zh.md`

- [ ] **Step 1: 英文 README 增加配置说明**

在 `README.md` 的 Configuration 代码块中（`cacheHashedAssets` 行之后）添加：

```yaml
    disableTokenAuth: false      # true: skip built-in token/session gate; external auth (e.g. Nginx + Authelia) is the only gate
```

并在 Configuration 段落的"Two points to note"列表后追加一段：

```markdown
- `disableTokenAuth: true` bypasses the built-in access-token / device-session /
  invite gate: every request is forwarded straight to the backend after CIDR and
  header checks. Use this only when an external reverse proxy (for example Nginx
  with the Authelia `auth_request` flow) already authenticated the client. The
  loopback-only control routes (`/dsh-reverse-proxy/*`), the health endpoint,
  CIDR allowlisting, header sanitization, gzip, hashed-asset caching, and the
  audit log still apply.
```

- [ ] **Step 2: 中文 README 同步更新**

在 `README.zh.md` 的 Configuration 代码块（`cacheHashedAssets` 行之后）添加：

```yaml
    disableTokenAuth: false      # true: 跳过内置令牌/会话门；认证完全由外部网关（如 Nginx + Authelia）负责
```

并在"Two points to note"列表后追加：

```markdown
- `disableTokenAuth: true` 会跳过内置的访问令牌 / 设备会话 / 邀请门：除 CIDR 与头部检查外，所有请求直接转发到后端。仅当外部反向代理（例如 Nginx + Authelia 的 `auth_request` 流程）已完成客户端认证时才应启用。回环专属控制路由（`/dsh-reverse-proxy/*`）、健康检查、CIDR 白名单、头部清洗、gzip、缓存与审计日志仍然生效。
```

- [ ] **Step 3: 提交**

```bash
git add README.md README.zh.md
git commit -m "docs: document disableTokenAuth for external auth deployments"
```

---

### Task 6: Nginx + Authelia 部署参考文档

**Files:**
- Create: `docs/nginx-authelia.md`
- Modify: `README.md`、`README.zh.md`（加文档链接）

**Interfaces:**
- Consumes: Task 5 的 README 配置说明
- Produces: 一份可直接参考的 Nginx + Authelia 部署文档

- [ ] **Step 1: 编写部署参考文档**

创建 `docs/nginx-authelia.md`，包含完整架构图、Nginx `server` 块配置（含 `auth_request` 指向 Authelia `/api/verify`、`error_page 401` 重定向、frp 转发）、以及安全提醒：

```markdown
# Nginx + Authelia 外部认证部署参考

配合 `dsh-full-remote` 的 `disableTokenAuth: true` 使用：由 Nginx + Authelia
在公网侧完成认证，NAS 上的代理只负责转发与头部改写，不再维护令牌。

## 架构

    远程浏览器 → Nginx (auth_request → Authelia)
                    ↓ 已认证
                 frp 服务端
                    ↓ frp 隧道
                 NAS: dsh-full-remote (disableTokenAuth: true)
                    ↓ Host/Origin 改写
                 DSH Web: 127.0.0.1:3080

## Nginx server 块参考

server {
    listen 443 ssl;
    server_name dsh.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # Authelia 验证端点（internal，禁止直接访问）
    location /authelia/ {
        internal;
        proxy_pass http://127.0.0.1:9091/api/verify;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
        proxy_set_header X-Original-Method $request_method;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        auth_request /authelia/;
        # 未认证时跳到 Authelia 登录页
        error_page 401 =302 https://auth.example.com/?rd=$scheme://$host$request_uri;

        proxy_pass http://127.0.0.1:8080;  # frp 服务端 HTTP vhost 端口（vhostHTTPPort，默认 8080；若用 TCP 隧道则填该隧道的 remotePort）
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

## 注意

- `auth_request` 使用 Nginx 子请求验证会话；Authelia 通过
  `X-Original-URL` 回跳，因此该头必须保持 `$scheme://$http_host$request_uri`。
- `error_page 401` 的 `?rd=` 参数是 Authelia 登录成功后的回跳地址；
  缺少它，用户登录后会被送回入口而非原本请求的页面。
- `proxy_pass` 指向 frp 服务端端口，frp 客户端把流量转发到 NAS 上
  dsh-full-remote 的监听地址（默认 127.0.0.1:3081）。
- WebSocket 需要 `Upgrade` / `Connection` 头透传，否则 Harness 的
  实时会话无法工作。
- 必须在 dsh-full-remote 端保持 `disableTokenAuth: true` 与回环
  `backendHost`；控制路由 `/dsh-reverse-proxy/*` 仍只允许本机访问。
- 若希望审计 / 限速看到真实客户端 IP，在 Nginx 侧保留
  `X-Forwarded-For` 并在 dsh-full-remote 开启 `trustForwardedFor: true`。
```

- [ ] **Step 2: README 添加文档链接**

在 `README.md` 的 Configuration 段落末尾（Task 5 追加的 `disableTokenAuth` 说明之后）添加：

```markdown
A deployment reference for Nginx + Authelia in front of the frp tunnel is in
[docs/nginx-authelia.md](./docs/nginx-authelia.md).
```

在 `README.zh.md` 对应位置添加：

```markdown
Nginx + Authelia 在 frp 隧道前的部署参考见 [docs/nginx-authelia.md](./docs/nginx-authelia.md)。
```

- [ ] **Step 3: 提交**

```bash
git add docs/nginx-authelia.md README.md README.zh.md
git commit -m "docs: add Nginx + Authelia deployment reference for disableTokenAuth"
```
