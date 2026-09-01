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
- 必须在 dsh-full-remote 端保持 `disableTokenAuth: true`，且代理的**监听地址
  `listenHost` 必须保持回环 `127.0.0.1`**（仅本机 frp 客户端 / 已完成认证的
  网关可达）。开启 `disableTokenAuth` 后令牌门被移除：若把 `listenHost` 配成
  局域网 IP（UI 引导的手机同 WiFi 直连场景），整个局域网都能无认证直达 DSH
  后端。`backendHost`（DSH Web 后端）本身已被校验强制回环，不是这里的风险点；
  控制路由 `/dsh-reverse-proxy/*` 仍只允许本机访问。
- 若希望 CIDR 白名单 / 审计 / 限速看到真实客户端 IP，**三个条件必须同时成立**：
  (1) Nginx 侧保留 `X-Forwarded-For $proxy_add_x_forwarded_for`；
  (2) frp 将该头原样转发到 NAS；
  (3) dsh-full-remote 开启 `trustForwardedFor: true`。
  否则所有客户端都会落到回环 `127.0.0.1`（回环恒被允许），CIDR 白名单将**静默失效**。
