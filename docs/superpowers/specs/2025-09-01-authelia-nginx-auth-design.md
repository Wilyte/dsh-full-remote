# Authelia + Nginx 外部认证集成设计

## 1. 背景

dsh-full-remote 当前内置完整的令牌 + 设备会话认证系统。用户希望将认证层外包给 **Authelia + Nginx**，由 Nginx 的 `auth_request` 指令在请求到达 frp 隧道之前完成认证，dsh-full-remote 不再需要维护自己的令牌体系。

## 2. 部署架构

```
远程浏览器 → Nginx (auth_request → Authelia)
                 ↓ (已认证)
              frp 服务端 (公网服务器)
                 ↓ (frp 隧道)
              NAS
                 ↓
              dsh-full-remote:3081 (令牌鉴权禁用)
                 ↓ (Host/Origin 改写)
              DSH Web:3080
```

- **Nginx + Authelia** 运行在公网服务器（frp 服务端）
- **dsh-full-remote** 运行在 NAS，令牌鉴权部分禁用，仅保留反向代理 + 头部改写 + 压缩功能
- **frp** 将公网服务器的流量隧道到 NAS 的 3081 端口

## 3. 变更范围

### 3.1 新增配置项 `disableTokenAuth`

在 `Config` 模式中新增布尔值选项，默认 `false`（保持向后兼容）：

```yaml
disableTokenAuth: false   # true 时跳过所有令牌/会话/邀请校验
```

当 `disableTokenAuth: true` 时，代理的行为变化：

| 行为 | 变化 |
|------|------|
| 令牌校验 | 跳过，所有请求直接放行 |
| 设备会话 | 跳过，不验证 cookie |
| 一次性邀请 | 跳过，`/login` 路由不再处理 |
| 设备首页 | 跳过，`/home`、`/logout` 等路由不再处理 |
| 等待审批页 | 跳过 |
| 登录限速 | 跳过 |
| Host/Origin 改写 | **保留** |
| 不安全头部剥离 | **保留** |
| gzip 压缩 | **保留** |
| 缓存控制 | **保留** |
| CIDR 白名单 | **保留**（可选，Nginx 端也可做一层） |
| 审计日志 | **保留**（记录请求来源，但无登录事件） |
| 控制路由 (`/dsh-reverse-proxy/*`) | **保留**（仍限 loopback） |
| 健康检查 (`/_dsh_reverse_proxy/healthz`) | **保留** |
| WebSocket 转发 | **保留**（跳过会话校验） |

### 3.2 修改的文件

#### `src/config.ts`
- 导入 `Schema` 后新增 `disableTokenAuth` 字段

#### `src/proxy.ts`
- `ProxySpec` 接口新增 `disableTokenAuth?: boolean`
- `RuntimeSpec` 添加对应字段
- `onRequest` 处理函数：当 `disableTokenAuth` 为 true 时，跳过 `LOGIN_PATH`、`waitRoutes`、`devicePages` 路由处理，跳过 `sessionStore.validate`，直接调用 `proxyRequest`
- `upgrade` 事件处理：当 `disableTokenAuth` 为 true 时，跳过会话校验直接转发
- `proxyRequest` 函数：`sessionId` 参数在无认证时传入 `'bypass'` 固定值

#### `src/index.ts`
- 在 `listenProxy` 调用时传入 `disableTokenAuth: config.disableTokenAuth`

#### `src/client/RemoteSection.tsx`
- 设置页展示当前认证模式，当 `disableTokenAuth` 开启时显示"外部认证（Authelia）"状态

### 3.3 不修改的文件

- `src/security.ts` — 安全原语不变，只是不被调用
- `src/sessions.ts` — 会话存储逻辑不变，只是不被调用
- `src/invites.ts` — 邀请逻辑不变，只是不被调用

## 4. 实现细节

### 4.1 `proxy.ts` 的 `onRequest` 流程变化

当前流程：
```
denyCidr → healthz → LOGIN_PATH → waitRoutes → devicePages → controlPrefix → validateSession → proxyRequest
```

当 `disableTokenAuth: true` 时变为：
```
denyCidr → healthz → [跳过 LOGIN_PATH] → [跳过 waitRoutes] → [跳过 devicePages] → controlPrefix → [跳过 validateSession] → proxyRequest
```

实现方式：在 `onRequest` 函数开头检查 `spec.disableTokenAuth`，若为 true 则跳过认证相关路由，直接进入 `proxyRequest`。

### 4.2 WebSocket 升级

当前流程：
```
CIDR 检查 → upgradeTracker 限速 → validateSession → 转发
```

当 `disableTokenAuth: true` 时：
```
CIDR 检查 → [跳过 upgradeTracker 限速] → [跳过 validateSession] → 转发
```

### 4.3 会话跟踪

当 `disableTokenAuth` 为 true 时，`proxyRequest` 的 `sessionId` 参数传入固定值 `'bypass'`。这不会影响 `trackSession` 功能（连接的跟踪/清理仍然正常工作），只是所有连接共享同一个虚拟会话 ID。

## 5. Nginx 配置参考

以下为推荐的 Nginx 配置（公网服务器上）：

```nginx
server {
    listen 443 ssl;
    server_name dsh.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Authelia 认证端点
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
        auth_request_set $authelia_url $upstream_http_location;

        # 未认证时重定向到 Authelia 登录
        error_page 401 =302 https://auth.example.com/?rd=$scheme://$host$request_uri;

        # 转发到 frp 服务端本地端口
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
```

## 6. 安全考虑

1. **认证层唯一性**：启用 `disableTokenAuth` 后，必须确保 Nginx + Authelia 层正确配置，否则任何请求都可直达 DSH 后端
2. **控制路由隔离**：`/dsh-reverse-proxy/*` 仍限 loopback，防止远程通过公网隧道修改代理配置
3. **审计日志**：建议保留审计日志，即使无令牌认证，仍可记录请求来源
4. **CIDR 白名单**：可在 dsh-full-remote 和 Nginx 两端同时配置，增加纵深防御
5. **TLS**：Nginx 端必须配置 TLS（已通过 ssl_certificate 确保），frp 隧道内流量建议加密

## 7. 向后兼容

- `disableTokenAuth` 默认 `false`，现有用户无感知
- 当 `disableTokenAuth: true` 时，`/login` 路径返回 404（或简单提示），避免混淆
- 所有已有功能保持不变，仅在认证入口处做条件判断