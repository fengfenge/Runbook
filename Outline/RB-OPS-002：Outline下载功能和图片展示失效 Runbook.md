# Outline-OPS-002：下载功能和图片展示失效 Runbook

> **Metadata**
>
> - 环境: 生产
>
> - 服务名称: `Outline`
> - 告警名称: `302 Found`
> - 负责人: @sre-oncall
> - 最后更新: 2026-08-04
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 用户反馈：Outline Web页面无法展示图片、下载功能失败

## 快速诊断

### 步骤1：查看开发者工具Network

```bash
outline中显示图片错误：Request URL
https://outline.tian-power.cloud/api/attachments.redirect?id=58bf62e6-3086-44be-af7a-b050fd4491c7
Request Method
GET
Status Code
302 Found (from disk cache)
Remote Address
192.168.13.23:443
Referrer Policy
no-referrer
```

### 步骤2：查看容器运行状态

```bash
# docker ps
CONTAINER ID   IMAGE                                      COMMAND                  CREATED         STATUS                   PORTS                                                                                      NAMES
7b5ec0edc477   outlinewiki/outline:1.9.2                  "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes (healthy)   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp                                                  outline_1.9.2-1
7d3abef791d1   keycloak/keycloak:26.1.4                   "/opt/bitnami/script…"   10 days ago     Up 10 days               8443/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 9000/tcp                              keycloak-1
a54b1beb1879   postgres:15.12                             "docker-entrypoint.s…"   4 months ago    Up 8 weeks               0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                                  postgres
3bbedc2c930d   minio/minio:RELEASE.2025-03-12T18-04-18Z   "/usr/bin/docker-ent…"   4 months ago    Up 8 weeks               0.0.0.0:29000->9000/tcp, :::29000->9000/tcp, 0.0.0.0:29001->9001/tcp, :::29001->9001/tcp   minio-cp
97782cbe3025   redis:7.4.2-alpine                         "docker-entrypoint.s…"   16 months ago   Up 8 weeks               0.0.0.0:6379->6379/tcp, :::6379->6379/tcp                                                  redis
```

### 步骤3：分析

```bash
docker run -d     -p 80:3000     --name outline_1.9.2    --network keycloak-network     -v ~/docker-data/outline-data:/var/lib/outline/data     -e DATABASE_URL=postgres://iotcloud:iotcloud@postgres/outline2     -e REDIS_URL=redis://:iotcloud@192.168.13.23:6379/0     -e PGSSLMODE=disable     -e FORCE_HTTPS=true    -e SECRET_KEY=22a3dbd6fd96864cb125504826b153cf60a454e43a019afe4692e06f6cacd700     -e UTILS_SECRET=11121c684ca0dc0352bf7567db9aaf9bc26c45fc34cacd5dbe883c086d7c7e00     -e URL=https://outline.tian-power.cloud     -e OIDC_CLIENT_ID=outline     -e OIDC_CLIENT_SECRET=y3WKtIycbXAYnmYu7zPxNpOcEy034EOn    -e OIDC_AUTH_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/auth     -e OIDC_TOKEN_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/token     -e OIDC_USERINFO_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/userinfo     -e OIDC_LOGOUT_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/logout?redirect_uri=http%3A%2F%2F192.168.13.23     -e OIDC_USERNAME_CLAIM=preferred_username     -e OIDC_DISPLAY_NAME=keycloak     -e OIDC_SCOPES="openid profile email"     -e AWS_ACCESS_KEY_ID=iotcloud     -e AWS_SECRET_ACCESS_KEY=iotcloud@2025     -e AWS_S3_UPLOAD_BUCKET_URL=http://192.168.13.23:29000     -e AWS_S3_UPLOAD_BUCKET_NAME=outline     -e AWS_REGION=cn-homelab-1     -e FILE_STORAGE_UPLOAD_MAX_SIZE=5621440000     -e AWS_S3_FORCE_PATH_STYLE=true     -e AWS_S3_ACL=private     -e SMTP_HOST=smtp.exmail.qq.com     -e SMTP_PORT=465     -e SMTP_USERNAME=iot-cloud@tian-power.com     -e SMTP_PASSWORD=Vb4Ca8H0fi3l6AE     -e SMTP_FROM_EMAIL=iot-cloud@tian-power.com     -e EMAIL_ENABLED=true -e GITLAB_CLIENT_ID=43c97e179744f685cd6528e35ad5352b1e9c37dcf9790679476a537635176c08 -e GITLAB_CLIENT_SECRET=gloas-988dd8d9d59aa6f6f0fd0c5e3a30f0d98e997752995174b4469ad83658d4adec -e GITLAB_BASE_URL=https://gitlab.tian-power.cloud -e DEFAULT_LANGUAGE=zh_CN -e WEB_CONCURRENCY=256 -e  ALLOWED_PRIVATE_IP_ADDRESSES=192.168.13.76  outlinewiki/outline:1.9.2

根据部署配置，问题很可能出在 AWS_S3_UPLOAD_BUCKET_URL 使用了内网 IP 地址。
Outline 的图片加载流程是：
浏览器请求 /api/attachments.redirect?id=...
Outline 验证用户身份（通过 Cookie/Session）后，返回 302 重定向 到实际的 S3 存储地址（即您配置的AWS_S3_UPLOAD_BUCKET_URL 加上签名参数）。
浏览器随后请求该重定向地址获取图片。
如果重定向后的地址是 http://192.168.13.23:29000/outline/...，
```

### 步骤4：解决办法

nginx做反向代理到minio容器

```bash
# vim minio.conf
server {
    listen 80;
    server_name minio.tian-power.cloud;
    # 如果需要 HTTP 重定向到 HTTPS，可以单独配置或在这里处理
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name minio.tian-power.cloud;

    # SSL 证书（请替换为您的实际证书路径）
    ssl_certificate     /etc/nginx/conf.d/ssl/tian-power.cloud.pem;
    ssl_certificate_key /etc/nginx/conf.d/ssl/tian-power.cloud.key;

    # SSL 安全设置（建议）
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # 日志（可选）
    access_log /data/wwwlog/minio_access.log;
    error_log  /data/wwwlog/minio_error.log;

    # 最大上传大小（与 Outline 的 FILE_STORAGE_UPLOAD_MAX_SIZE 保持一致）
    client_max_body_size 5G;

    location / {
        # 代理到 MinIO 服务（内网地址）
        proxy_pass http://192.168.13.23:29000;

        # 传递必要的头部
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 禁用缓存（可选）
        proxy_buffering off;

        # 超时设置（大文件上传可能需要）
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;
    }
}
```

outline启动命令参数修改

```bash
# 修改变量AWS_S3_UPLOAD_BUCKET_URL=https://minio.tian-power.cloud 和OIDC_LOGOUT_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/logout?redirect_uri=http%3A%2F%2Foutline.tian-power.cloud
docker run -d     -p 3000:3000     --name outline_1.9.2-1    --network keycloak-network     -v ~/docker-data/outline-data:/var/lib/outline/data     -e DATABASE_URL=postgres://iotcloud:iotcloud@postgres/outline2     -e REDIS_URL=redis://:iotcloud@192.168.13.23:6379/0     -e PGSSLMODE=disable     -e FORCE_HTTPS=true    -e SECRET_KEY=22a3dbd6fd96864cb125504826b153cf60a454e43a019afe4692e06f6cacd700     -e UTILS_SECRET=11121c684ca0dc0352bf7567db9aaf9bc26c45fc34cacd5dbe883c086d7c7e00     -e URL=https://outline.tian-power.cloud     -e OIDC_CLIENT_ID=outline     -e OIDC_CLIENT_SECRET=y3WKtIycbXAYnmYu7zPxNpOcEy034EOn    -e OIDC_AUTH_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/auth     -e OIDC_TOKEN_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/token     -e OIDC_USERINFO_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/userinfo     -e OIDC_LOGOUT_URI=http://192.168.13.23:8080/realms/outline/protocol/openid-connect/logout?redirect_uri=http%3A%2F%2Foutline.tian-power.cloud     -e OIDC_USERNAME_CLAIM=preferred_username     -e OIDC_DISPLAY_NAME=keycloak     -e OIDC_SCOPES="openid profile email"     -e AWS_ACCESS_KEY_ID=iotcloud     -e AWS_SECRET_ACCESS_KEY=iotcloud@2025     -e AWS_S3_UPLOAD_BUCKET_URL=https://minio.tian-power.cloud     -e AWS_S3_UPLOAD_BUCKET_NAME=outline     -e AWS_REGION=cn-homelab-1     -e FILE_STORAGE_UPLOAD_MAX_SIZE=5621440000     -e AWS_S3_FORCE_PATH_STYLE=true     -e AWS_S3_ACL=private     -e SMTP_HOST=smtp.exmail.qq.com     -e SMTP_PORT=465     -e SMTP_USERNAME=iot-cloud@tian-power.com     -e SMTP_PASSWORD=Vb4Ca8H0fi3l6AE     -e SMTP_FROM_EMAIL=iot-cloud@tian-power.com     -e EMAIL_ENABLED=true -e GITLAB_CLIENT_ID=43c97e179744f685cd6528e35ad5352b1e9c37dcf9790679476a537635176c08 -e GITLAB_CLIENT_SECRET=gloas-988dd8d9d59aa6f6f0fd0c5e3a30f0d98e997752995174b4469ad83658d4adec -e GITLAB_BASE_URL=https://gitlab.tian-power.cloud -e DEFAULT_LANGUAGE=zh_CN -e WEB_CONCURRENCY=256 -e  ALLOWED_PRIVATE_IP_ADDRESSES=192.168.13.76  outlinewiki/outline:1.9.2
```

### 步骤5：验证

```bash
outline.tian-power.cloud登录
检查浏览器开发者工具的 Network 面板：
点击图片请求，查看重定向后的 URL 是否变成了可访问的域名。
1、图片正常展示
2、文件可正常下载
3、上传图片和文件正常
4、docker logs --tail 10 <outline_container>查看日志无报错
```

