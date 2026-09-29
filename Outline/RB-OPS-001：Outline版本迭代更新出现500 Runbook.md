# Outline-OPS-001：版本迭代更新出现500 Runbook

> **Metadata**
>
> - 环境: 生产
>
> - 服务名称: `Outline`
> - 告警名称: `500`
> - 负责人: @sre-oncall
> - 最后更新: 2026-08-04
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 用户反馈：Outline Web页面无法报500错误码

## 快速诊断

### 步骤1: 查看容器运行状态

```bash
# docker ps 
CONTAINER ID   IMAGE                                      COMMAND                  CREATED         STATUS                   PORTS                                                                                      NAMES
7b5ec0edc477   outlinewiki/outline:1.9.2                  "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes (healthy)   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp                                                  outline_1.9.2-1
7d3abef791d1   keycloak/keycloak:26.1.4                   "/opt/bitnami/script…"   10 days ago     Up 10 days               8443/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 9000/tcp                              keycloak-1
a54b1beb1879   postgres:15.12                             "docker-entrypoint.s…"   4 months ago    Up 8 weeks               0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                                  postgres
3bbedc2c930d   minio/minio:RELEASE.2025-03-12T18-04-18Z   "/usr/bin/docker-ent…"   4 months ago    Up 8 weeks               0.0.0.0:29000->9000/tcp, :::29000->9000/tcp, 0.0.0.0:29001->9001/tcp, :::29001->9001/tcp   minio-cp
97782cbe3025   redis:7.4.2-alpine                         "docker-entrypoint.s…"   16 months ago   Up 8 weeks               0.0.0.0:6379->6379/tcp, :::6379->6379/tcp                                                  redis
# 依赖的服务status全部正常
```

### 步骤2: 查看outline容器日志

```bash
# docker logs --tail 33 <outline_container>
{"label":"lifecycle","level":"info","message":"Listening on http://localhost:3000 / http://192.168.13.23"}
Error: Cannot send secure cookie over unencrypted connection
    at Cookies.set (/opt/outline/node_modules/cookies/index.js:126:11)
    at StateStore.store (/opt/outline/build/server/utils/passport.js:126:23)
    at OAuth2Strategy.authenticate (/opt/outline/node_modules/passport-oauth2/lib/strategy.js:291:28)
    at OIDCStrategy.authenticate (/opt/outline/build/plugins/oidc/server/auth/OIDCStrategy.js:19:11)
    at attempt (/opt/outline/node_modules/@outlinewiki/koa-passport/node_modules/passport/lib/middleware/authenticate.js:369:16)
    at authenticate (/opt/outline/node_modules/@outlinewiki/koa-passport/node_modules/passport/lib/middleware/authenticate.js:370:7)
    at /opt/outline/node_modules/@outlinewiki/koa-passport/lib/framework/koa.js:194:7
    at new Promise (<anonymous>)
    at /opt/outline/node_modules/@outlinewiki/koa-passport/lib/framework/koa.js:193:12
    at /opt/outline/node_modules/@outlinewiki/koa-passport/lib/framework/koa.js:143:7
    at new Promise (<anonymous>)
    at passportAuthenticate (/opt/outline/node_modules/@outlinewiki/koa-passport/lib/framework/koa.js:107:15)
    at passportAuthenticate (/opt/outline/node_modules/dd-trace/packages/datadog-instrumentations/src/koa.js:90:57)
    at dispatch (/opt/outline/node_modules/koa-router/node_modules/koa-compose/index.js:44:32)
    at next (/opt/outline/node_modules/koa-router/node_modules/koa-compose/index.js:45:18)
    at startOAuthFlow (/opt/outline/build/server/utils/passport.js:56:12)
```

### 步骤3: 日志分析

```bash
Outline 的 OIDC 登录流程需要设置一个 secure cookie（用于存储 OAuth state），而 secure cookie 只能通过 HTTPS 发送，不能通过 HTTP 发送。
从日志看，Outline 当前运行在 http://192.168.13.23（明文 HTTP），所以触发了这个报错。

Outline 1.8.x 在 production 模式下，OIDC 认证流程中的 state cookie（用于防 CSRF）代码里写死了 secure: _env.default.isProduction，即 secure: true。
而访问方式是纯 HTTP (http://192.168.13.23)，Node.js 的 cookies 库会拒绝在 HTTP 连接上发送 secure cookie，所以直接抛错：
Error: Cannot send secure cookie over unencrypted connection
```

```bash
分析容器运行的变量，必须使用https协议，变更变量的值
# docker run -d     -p 80:3000     --name outline_1.8.2-0    --network keycloak-network     -v ~/docker-data/outline-data:/var/lib/outline/data     -e DATABASE_URL=postgres://iotcloud:iotcloud@postgres/outline2     -e REDIS_URL=redis://:iotcloud@192.168.13.23:6379/0     -e PGSSLMODE=disable     -e FORCE_HTTPS=false     -e SECRET_KEY=22a3dbd6fd96864cb125504826b153cf60a454e43a019afe4692e06f6cacd700     -e UTILS_SECRET=11121c684ca0dc0352bf7567db9aaf9bc26c45fc34cacd5dbe883c086d7c7e00     -e URL=http://192.168.13.23     -e OIDC_CLIENT_ID=outline     -e OIDC_CLIENT_SECRET=y3WKtIycbXAYnmYu7zPxNpOcEy034EOn    -e OIDC_AUTH_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/auth     -e OIDC_TOKEN_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/token     -e OIDC_USERINFO_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/userinfo     -e OIDC_LOGOUT_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/logout?redirect_uri=http%3A%2F%2F192.168.13.23     -e OIDC_USERNAME_CLAIM=preferred_username     -e OIDC_DISPLAY_NAME=keycloak     -e OIDC_SCOPES="openid profile email"     -e AWS_ACCESS_KEY_ID=iotcloud     -e AWS_SECRET_ACCESS_KEY=iotcloud@2025     -e AWS_S3_UPLOAD_BUCKET_URL=http://192.168.13.23:29000     -e AWS_S3_UPLOAD_BUCKET_NAME=outline     -e AWS_REGION=cn-homelab-1     -e FILE_STORAGE_UPLOAD_MAX_SIZE=5621440000     -e AWS_S3_FORCE_PATH_STYLE=true     -e AWS_S3_ACL=private     -e SMTP_HOST=smtp.exmail.qq.com     -e SMTP_PORT=465     -e SMTP_USERNAME=iot-cloud@tian-power.com     -e SMTP_PASSWORD=Vb4Ca8H0fi3l6AE     -e SMTP_FROM_EMAIL=iot-cloud@tian-power.com     -e EMAIL_ENABLED=true -e GITLAB_CLIENT_ID=43c97e179744f685cd6528e35ad5352b1e9c37dcf9790679476a537635176c08 -e GITLAB_CLIENT_SECRET=gloas-988dd8d9d59aa6f6f0fd0c5e3a30f0d98e997752995174b4469ad83658d4adec -e GITLAB_BASE_URL=https://gitlab.tian-power.cloud -e DEFAULT_LANGUAGE=zh_CN -e WEB_CONCURRENCY=256 -e  ALLOWED_PRIVATE_IP_ADDRESSES=192.168.13.76  outlinewiki/outline:1.8.2

#现使用的是http协议
FORCE_HTTPS=false
URL=http://192.168.13.23
```

### 步骤4: 解决方案

nginx web服务作为反向代理

```bash
# 使用nginx web服务作为反向代理到容器启动的outline服务
# vim outline-web.conf 

server {
    listen 80;
    server_name outline.tian-power.cloud;

    # 自动重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name outline.tian-power.cloud;

    ssl_certificate /etc/nginx/conf.d/ssl/tian-power.cloud.pem;
    ssl_certificate_key  /etc/nginx/conf.d/ssl/tian-power.cloud.key;

    access_log /data/wwwlog/web-access.log;
    error_log /data/wwwlog/web-error.log;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {

        proxy_http_version 1.1;
        # 关键：必须传递这些头，让 Outline 知道实际是 HTTPS
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;

        # WebSocket 支持（Outline 实时协作需要）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 超时设置
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;

        proxy_pass http://127.0.0.1:3000;
        proxy_connect_timeout 20s;
    }
}
```

修改outline容器启动方式

```bash
# 主要针对的是FORCE_HTTPS=true和URL=https://outline.tian-power.cloud协议域名指定
docker run -d     -p 80:3000     --name outline_1.9.2    --network keycloak-network     -v ~/docker-data/outline-data:/var/lib/outline/data     -e DATABASE_URL=postgres://iotcloud:iotcloud@postgres/outline2     -e REDIS_URL=redis://:iotcloud@192.168.13.23:6379/0     -e PGSSLMODE=disable     -e FORCE_HTTPS=true    -e SECRET_KEY=22a3dbd6fd96864cb125504826b153cf60a454e43a019afe4692e06f6cacd700     -e UTILS_SECRET=11121c684ca0dc0352bf7567db9aaf9bc26c45fc34cacd5dbe883c086d7c7e00     -e URL=https://outline.tian-power.cloud     -e OIDC_CLIENT_ID=outline     -e OIDC_CLIENT_SECRET=y3WKtIycbXAYnmYu7zPxNpOcEy034EOn    -e OIDC_AUTH_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/auth     -e OIDC_TOKEN_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/token     -e OIDC_USERINFO_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/userinfo     -e OIDC_LOGOUT_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/logout?redirect_uri=http%3A%2F%2F192.168.13.23     -e OIDC_USERNAME_CLAIM=preferred_username     -e OIDC_DISPLAY_NAME=keycloak     -e OIDC_SCOPES="openid profile email"     -e AWS_ACCESS_KEY_ID=iotcloud     -e AWS_SECRET_ACCESS_KEY=iotcloud@2025     -e AWS_S3_UPLOAD_BUCKET_URL=http://192.168.13.23:29000     -e AWS_S3_UPLOAD_BUCKET_NAME=outline     -e AWS_REGION=cn-homelab-1     -e FILE_STORAGE_UPLOAD_MAX_SIZE=5621440000     -e AWS_S3_FORCE_PATH_STYLE=true     -e AWS_S3_ACL=private     -e SMTP_HOST=smtp.exmail.qq.com     -e SMTP_PORT=465     -e SMTP_USERNAME=iot-cloud@tian-power.com     -e SMTP_PASSWORD=Vb4Ca8H0fi3l6AE     -e SMTP_FROM_EMAIL=iot-cloud@tian-power.com     -e EMAIL_ENABLED=true -e GITLAB_CLIENT_ID=43c97e179744f685cd6528e35ad5352b1e9c37dcf9790679476a537635176c08 -e GITLAB_CLIENT_SECRET=gloas-988dd8d9d59aa6f6f0fd0c5e3a30f0d98e997752995174b4469ad83658d4adec -e GITLAB_BASE_URL=https://gitlab.tian-power.cloud -e DEFAULT_LANGUAGE=zh_CN -e WEB_CONCURRENCY=256 -e  ALLOWED_PRIVATE_IP_ADDRESSES=192.168.13.76  outlinewiki/outline:1.9.2
```

### 步骤5: 登录outline

进入Keycloak登录界面报错：Invalid parameter: redirect_uri

分析得到**Keycloak 中 Outline 客户端的 `Valid Redirect URIs` 没有包含 `https://outline.tian-power.cloud/auth/oidc.callback`**

### 步骤6: Keycloak修复

1. 找到 Outline 客户端

- 进入你的 Realm（如 `outline`）
- 左侧菜单 → **Clients** → 点击 `outline`

2. 修改 Valid Redirect URIs

- Root URL：https://outline.tian-power.cloud/
- Home URL：https://outline.tian-power.cloud/
- Valid redirect URIs：https://outline.tian-power.cloud/auth/oidc.callback
- Valid post logout redirect URIs：https://outline.tian-power.cloud/*
- Web origins：https://outline.tian-power.cloud
- Admin URL：https://outline.tian-power.cloud/

### 步骤7: 验证

```bash
https://outline.tian-power.cloud域名登录，正常显示
# 查看outline容器日志，无报错
docker logs --tail 33 <outline_container>
```

