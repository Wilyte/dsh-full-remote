# Nginx + Authelia 外部认证

> [English](./external-auth.md)

本指南完整讲解如何把认证委托给已有的 Nginx + Authelia（或类似 SSO）边缘，
让 `dsh-full-remote` 不再维护自己的访问令牌 / 设备会话 / 邀请体系。紧凑的
部署参考见 [nginx-authelia.md](./nginx-authelia.md)；本页是完整配置指南——
逐项配置说明 + 验证步骤。

## 1. 何时使用此模式

当**以下条件全部满足**时使用 `disableTokenAuth: true`：

- 组织里已经有统一认证网关（典型是 Nginx + Authelia 的 `auth_request`
  流程，或其他 SSO）；
- 到达 NAS 的每个请求都已经过该网关——无论网关本身是公网入口，还是你的
  frp / 隧道只接收已认证流量；
- 你不想仅为 Harness 再维护一套凭据体系（令牌轮换、设备会话、邀请）。

开启后，代理保留它存在的意义——`Host`/`Origin` 改写、头部清洗、CIDR
白名单、gzip、带 hash 资源缓存、健康检查、回环专属控制路由——但**完全
停止校验凭据**。网关成为唯一闸门。

## 2. 架构

```
远程浏览器 ──> Nginx (auth_request ──> Authelia)
                 │  已认证？
                 ▼
              frp 服务端（公网主机）
                 │  frp 隧道
                 ▼
              NAS: dsh-full-remote  (disableTokenAuth: true)
                 │  Host/Origin 改写 + 头部清洗
                 ▼
              DSH Web: 127.0.0.1:3080
```

浏览器到 NAS 之间的每一跳都必须被网关认证覆盖——若没有覆盖，失败模式见
[安全注意](#8-安全注意)。

## 3. Nginx 配置（公网服务器）

完整的 `server` 块。两个 `location` 块是 Authelia 集成的核心，其余是
标准反向代理管道。

```nginx
server {
    listen 443 ssl;
    server_name dsh.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # ── Authelia 验证端点 ─────────────────────────────────────────────
    # Nginx 子请求目标。`internal` 阻止浏览器直接访问。
    location /authelia/ {
        internal;
        proxy_pass http://127.0.0.1:9091/api/verify;
        proxy_set_header Content-Length "";                      # 子请求按 GET 处理
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;   # 用户原本要去哪
        proxy_set_header X-Original-Method $request_method;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # ── 其余所有请求 ─────────────────────────────────────────────────
    location / {
        auth_request /authelia/;
        # 未认证 → Authelia 登录页；?rd= 登录成功后回到原本请求的页面。
        # 用 $host（而非 $http_host）避免 Host 头注入。
        error_page 401 =302 https://auth.example.com/?rd=$scheme://$host$request_uri;

        # frp 服务端 HTTP vhost 端口（vhostHTTPPort，默认 8080）；
        # 若用普通 TCP 隧道，则填该隧道的 remotePort。
        proxy_pass http://127.0.0.1:8080;

        proxy_http_version 1.1;
        # WebSocket 支持——Harness 实时会话必需：
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Authelia 集成要点：

- **`auth_request /authelia/`**：Nginx 对每个请求向 `location /authelia/`
  发一个子请求。Authelia 应答 `200`（会话有效）或 `401`（转登录）。
- **`X-Original-URL`** 必须保持 `$scheme://$http_host$request_uri` 形式——
  Authelia 靠它判断用户原本要去哪。
- **`error_page 401 =302 .../?rd=...`**：`?rd=` 是登录成功后的回跳地址。
  缺少它，用户会落在 Authelia 门户而不是请求的页面。用 `$host`（而非
  `$http_host`）防止恶意 `Host` 头劫持跳转。
- **WebSocket**：`Upgrade` / `Connection` 头透传是必须的，否则 Harness
  实时会话无法工作。

## 4. frp 隧道配置

| 问题 | 答案 |
|---|---|
| Nginx 的 `proxy_pass` 指向哪里？ | frp **服务端**的 HTTP vhost 端口——`vhostHTTPPort`，默认 `8080`。若跑普通 TCP 隧道，则指向该隧道的 `remotePort`。 |
| frp 客户端转发到哪里？ | NAS 上 `dsh-full-remote` 的监听地址——保持 `listenHost: 127.0.0.1`、`listenPort: 3081`。 |
| frp 会转发头部吗？ | 只有配置了才转发。`X-Forwarded-For` 必须原样存活到 NAS，真实客户端 IP 功能才可用（见下）。 |

关键：**`X-Forwarded-For` 透传**。要让插件看到真实客户端 IP，**三个条件
必须同时成立**：

1. Nginx 发送 `X-Forwarded-For $proxy_add_x_forwarded_for`（上文已含）；
2. frp 把该头原样转发到 NAS；
3. `dsh-full-remote` 开启 `trustForwardedFor: true`。

否则所有客户端在代理处塌缩为 `127.0.0.1`——而 CIDR 白名单**恒放行回环**——
于是 `allowedCidrs` 静默失效，审计/限速也只会看到一个假 IP。

## 5. dsh-full-remote 配置

安装插件后，在 profile 的补丁层设置 `disableTokenAuth: true`。web profile：

```yaml
# /Users/<你>/.dsh/profiles/web/cordis.patch.yml
- id: reverse-proxy
  name: dsh-full-remote
  config:
    disableTokenAuth: true
```

### 逐项配置参考

| 选项 | 默认 | 外部认证模式下 |
|---|---|---|
| `disableTokenAuth` | `false` | **设为 `true`**。整体跳过令牌 / 会话 / 邀请门。 |
| `listenHost` | `127.0.0.1` | **必须保持 `127.0.0.1`**（回环）——见安全注意。 |
| `listenPort` | `3081` | 保持，或改成 frp 转发的目标端口。 |
| `backendHost` | `127.0.0.1` | 保持回环——校验已强制。 |
| `trustForwardedFor` | `false` | 仅当 Nginx 保留 `X-Forwarded-For` **且** frp 原样转发时设为 `true`（上述三条件）。 |
| `trustCloudflareConnectingIp` | `false` | 仅本地 Cloudflare 连接器，且必须与 `trustForwardedFor: true` 同开。 |
| `allowedCidrs` | `[]` | 可选纵深防御。仅当三条件成立时才对真实客户端 IP 生效。 |
| `logRequests` | `false` | 排查有用：记录每个被代理的请求。 |
| `auditLog` | `true` | 保持开启——请求仍被审计（无登录事件，但访问/拒绝事件保留）。 |
| `compressResponses` / `cacheHashedAssets` | `true` | 不受此模式影响。 |

其余选项（超时、TLS、`approvalMode`、会话设置、邀请设置）在
`disableTokenAuth: true` 下不生效——使用它们的闸门已被跳过。

## 6. 验证

重启 `dsh web` 后：

1. **面板徽标**——设置 → 反向代理在 `disableTokenAuth` 开启时显示
   「外部认证模式」提示。
2. **无需令牌**——经代理的匿名请求（如 `curl http://127.0.0.1:3081/`）
   直达 DSH 后端。没有登录跳转、没有 cookie 校验。
3. **控制路由仍被拦截**——`curl http://127.0.0.1:3081/dsh-reverse-proxy/status`
   返回 `403`。
4. **健康检查正常**——`curl http://127.0.0.1:3081/_dsh_reverse_proxy/healthz`
   返回 `{"ok":true}`。
5. **头部改写完好**——开启 `logRequests: true` 后，调试日志显示请求被
   转发到 `127.0.0.1:<backendPort>`。
6. **端到端**——用无痕窗口打开 Nginx 网关地址：被送到 Authelia，登录后
   落到 DSH 页面，设置 / 凭据 / 目录浏览全部可用（无 403）。

## 7. 切回内置认证

把 `disableTokenAuth` 设为 `false`（或删掉该行）并重启即可恢复内置闸门。
幸存的令牌、会话、邀请照常工作——开关此选项不会改动代理本身。

## 8. 安全注意

- **暴露向量是 `listenHost`，不是 `backendHost`。** `disableTokenAuth: true`
  后代理不再校验任何凭据。若 `listenHost` 是局域网 IP（UI 引导的手机同
  WiFi 场景），整个局域网都能无认证直达 DSH 后端——`allowedCidrs` 默认为
  空，没有任何兜底。保持 `listenHost: 127.0.0.1`，只让代理经已认证的
  网关/隧道可达。
- **隧道下 `allowedCidrs` 会静默失效**，除非第 4 节三条件成立。回环恒被
  放行，塌缩到 `127.0.0.1` 的客户端会绕过任何 CIDR 列表。
- **网关是唯一闸门。** Nginx 配置错误（某路径未开认证、或端口裸奔）会
  直接暴露整个 DSH 后端。请用一个没有会话的浏览器验证完整路径。
- **TLS** 在 Nginx 终结；frp 一跳与 NAS 监听视为内网。控制路由保持
  回环专属——公网代理无法到达它们。
