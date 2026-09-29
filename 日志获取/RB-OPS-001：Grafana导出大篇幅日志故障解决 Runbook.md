# Loki-OPS-001：Grafana导出大篇幅日志故障解决 Runbook

> **Metadata**
>
> - 环境: 生产
>
> - 服务名称: `kuboard`
> - 告警名称: `连接异常`
> - 负责人: @sre-oncall
> - 最后更新: 2026-07-30
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 用户反馈：Grafana无法获取大篇幅日志，只能每次获取5000条日志，无法获取15：40-16：00时间段内全部的日志（超过几十万条）

## 前置检查清单

- [x] 已连接Grafana
- [x] 进入部署的Loki服务
- [x] 已通知值班群：`「<service-name> 排查中，@oncall」`



## 快速诊断

### 步骤1：查看日志系统Loki配置

```bash
# 默认每次获取日志数据是5000条
# vim /etc/loki/config.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: debug
  grpc_server_max_concurrent_streams: 1000

common:
  instance_addr: 127.0.0.1
  path_prefix: /opt/data/loki-data
  storage:
    filesystem:
      chunks_directory: /opt/data/loki-data/chunks
      rules_directory: /opt/data/loki-data/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

limits_config:
  metric_aggregation_enabled: true
  enable_multi_variant_queries: true
  reject_old_samples: true
  reject_old_samples_max_age: 168h         # 14天前的数据拒绝收集
  retention_period: 1440h                  # 30天：数据保留期
  max_query_lookback: 1440h                # 30天：最大查询回溯
  # max_entries_limit_per_query: 250000      # 将单次查询结果上限调整为250000条
  # query_timeout: 10m  # 设为10分钟甚至更长

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
        

pattern_ingester:
  enabled: true
  metric_aggregation:
    loki_address: localhost:3100

ruler:
  alertmanager_url: http://localhost:9093

frontend:
  encoding: protobuf
```

### 步骤2：使用logcli导出日志

```bash
# 获取大篇幅日志导入到文件中
logcli query '{app="general-consumer-public"}' --from="2026-08-06T15:30:00+08:00" --to="2026-08-06T16:00:00+08:00" --limit=1000000 --batch=5000 --forward --output=default > loki-1530-1600.log

# 将文件上传到oss桶中
#!/bin/bash
ACCESS_KEY_ID="<YOUR_ACCESS_KEY_ID>"
ACCESS_KEY_SECRET="<YOUR_ACCESS_KEY_SECRET>"
BUCKET="ops-inspection-report"
ENDPOINT="oss-cn-shenzhen.aliyuncs.com"
OBJECT="loki/loki-1530-1600.log"
FILE="./loki-1530-1600.log"
CONTENT_TYPE="application/octet-stream"
DATE=$(TZ=GMT date "+%a, %d %b %Y %H:%M:%S GMT")
RESOURCE="/${BUCKET}/${OBJECT}"
STRING_TO_SIGN="PUT\n\n${CONTENT_TYPE}\n${DATE}\n${RESOURCE}"
SIGNATURE=$(printf "%b" "${STRING_TO_SIGN}" | openssl dgst -sha1 -hmac "${ACCESS_KEY_SECRET}" -binary | base64)
curl -X PUT "https://${BUCKET}.${ENDPOINT}/${OBJECT}" \
  -H "Date: ${DATE}" \
  -H "Content-Type: ${CONTENT_TYPE}" \
  -H "Authorization: OSS ${ACCESS_KEY_ID}:${SIGNATURE}" \
  --data-binary @"${FILE}" \
  -w "\nHTTP %{http_code}\n"
```

