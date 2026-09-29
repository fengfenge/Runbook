# RB-OPS-002：Redis 缓存雪崩故障处理 Runbook

> **元数据**
> - 文档名称: Redis 缓存雪崩
> - 适用场景: 大量缓存同时过期 / Redis 集群故障导致请求直接打到数据库
> - 负责人: @sre-oncall / @cache-team
> - 最后更新: 2026-07-16
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P2（部分接口慢）/ P1（数据库被打挂）

## 触发条件

- [ ] 监控告警：`redis_cache_hit_rate < 50%` 突降
- [ ] 监控告警：`db_qps` 与 `redis_qps` 成反比飙升
- [ ] 数据库 CPU / 连接数突增，伴随 Redis 节点异常
- [ ] 应用日志出现大量：`Cache miss`、`RedisConnectionFailureException`
- [ ] 特定时间点（如整点缓存批量过期）接口集体变慢
- [ ] Redis 集群主从切换 / 节点宕机后数据库压力骤增

---

## 前置检查清单（1 分钟）

- [ ] 已连接生产 VPN / 跳板机
- [ ] `kubectl` 上下文已切换至 `prod` 集群
- [ ] 已打开 Redis 监控：
- [ ] 已通知值班群：`「Redis 缓存雪崩，排查中」`
- [ ] 已确认数据库当前负载（防止雪崩导致 DB 连锁故障）

---

## 快速诊断

### 步骤 1：确认 Redis 集群状态（1 分钟）

```bash
# 登录 Redis 集群（通过 redis-cli 或 Pod）
redis-cli -h <redis-cluster-endpoint> -p 6379 -c

# 查看集群节点状态
CLUSTER NODES

# 查看主从复制状态
INFO replication

# 查看内存使用
INFO memory
```

**关键判断**：

| 指标                        | 正常      | 异常   | 含义                        |
| :-------------------------- | :-------- | :----- | :-------------------------- |
| `cluster_state`             | `ok`      | `fail` | 集群状态异常                |
| `connected_slaves`          | >= 1      | 0      | 主库无从库，容灾能力下降    |
| `used_memory`               | < 80% max | > 90%  | 内存不足，可能触发驱逐      |
| `evicted_keys`              | 0 或低    | 突增   | 大量 Key 被驱逐（类似雪崩） |
| `instantaneous_ops_per_sec` | 平稳      | 骤降   | Redis 处理能力下降          |

------

### 步骤 2：确认缓存命中率（1 分钟）

```bash
# 在应用 Pod 内查看缓存指标（如 Actuator）
curl -s http://localhost:8080/actuator/metrics/cache.hit
curl -s http://localhost:8080/actuator/metrics/cache.miss

# 或查看应用日志
kubectl logs <pod-name> -n prod --tail=1000 | grep -c "Cache miss"
```

**判断**：

- 缓存命中率从 90%+ 降至 50% 以下 → **确认雪崩**
- 大量 `Cache miss` 同时出现 → **批量 Key 同时过期**

------

### 步骤 3：检查 Key 过期模式（2 分钟）

```bash
# 在 Redis 中检查大量 Key 的 TTL
redis-cli --scan --pattern "user:*" | head -20 | xargs -I {} redis-cli TTL {}

# 或检查某个热点 Key 的过期时间
redis-cli TTL <hot-key>

# 查看过期键统计
redis-cli INFO keyspace
```

**常见雪崩模式**：

| 模式           | 特征                                | 根因                            |
| :------------- | :---------------------------------- | :------------------------------ |
| 整点批量过期   | 大量 Key TTL 集中在同一秒           | 缓存设置时未加随机偏移          |
| 热点 Key 过期  | 单个 Key 访问量极大，过期后瞬间击穿 | 热点 Key 未设置永不过期或互斥锁 |
| Redis 节点宕机 | 某个 Slot 完全不可用                | 集群故障导致大面积 miss         |
| 内存驱逐       | `evicted_keys` 突增                 | 内存配置不足，Redis 主动驱逐    |

------

### 步骤 4：检查数据库负载（1 分钟）

```bash
# 查看数据库当前 QPS 和活跃连接
mysql -h <db-host> -u readonly -p -e "SHOW STATUS LIKE 'Threads_connected'; SHOW STATUS LIKE 'Queries';"

# 或从监控确认 DB CPU / 连接数是否飙升
```

**判断**：

- DB QPS 与 Redis Miss 同步飙升 → **确认雪崩打到 DB**
- DB 连接数接近上限 → **立即限流 + 回源保护**

------

## 处置措施

### 措施 A：紧急限流（保护数据库）

```bash
# 方式1：API Gateway 限流
curl -X POST http://gateway/api/rate-limit \
  -d '{"service":"<service-name>","qps":50,"burst":10}'

# 方式2：Nginx 限流（如通过 Nginx 代理）
# 在 Nginx 配置中临时添加：
# limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

# 方式3：应用层限流（Sentinel / Hystrix）
# 通过配置中心开启热点接口限流
curl -X POST http://config-center/api/switch \
  -d '{"service":"<service-name>","limiter":"emergency","qps":30}'
```

------

### 措施 B：快速重建缓存（互斥锁 + 异步加载）

```bash
# 如果应用支持，通过配置中心开启：
# 1. 热点 Key 互斥锁（防止并发回源）
curl -X POST http://config-center/api/switch \
  -d '{"service":"<service-name>","cache.lock":"true","lock.timeout":"10s"}'

# 2. 缓存预热接口手动触发
curl -X POST http://<service-name>/api/cache/warmup \
  -H "Authorization: Bearer <token>" \
  -d '{"keys":["user:*","product:*"]}'
```

------

### 措施 C：Redis 内存扩容（如内存不足导致驱逐）

```bash
# 阿里云 Redis 控制台扩容
# AWS ElastiCache 修改节点类型
# 或自建 Redis 增加 maxmemory

# 临时调整淘汰策略（避免继续驱逐）
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

------

### 措施 D：数据库连接池扩容（临时应对回源压力）

```bash
# 临时扩容 DB 连接池（同 RB-OPS-002）
kubectl patch deployment <service-name> -n prod -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<service-name>",
          "env": [
            {"name": "DB_MAX_POOL_SIZE", "value": "100"}
          ]
        }]
      }
    }
  }
}'
```

------

### 措施 E：降级非核心功能



```bash
# 关闭依赖缓存的非核心接口
curl -X POST http://config-center/api/switch \
  -d '{"service":"<service-name>","features":["recommendation","statistics"],"enabled":"false"}'
```

------

## 缓存雪崩预防配置（事后修复）

| 配置项             | 推荐方案                    | 说明                        |
| :----------------- | :-------------------------- | :-------------------------- |
| 过期时间加随机偏移 | `TTL = base + rand(0, 300)` | 避免整点批量过期            |
| 热点 Key 永不过期  | 设置 `NX` + 定时异步刷新    | 物理删除时主动重建          |
| 互斥锁回源         | `SET lock:<key> EX 10 NX`   | 防止并发穿透到 DB           |
| 多级缓存           | Caffeine (L1) + Redis (L2)  | 即使 Redis 故障仍有本地缓存 |
| 熔断降级           | Sentinel / Hystrix          | Redis 异常时直接返回默认值  |
| 缓存预热           | 发布时主动加载热点数据      | 避免冷启动瞬间击穿          |

------

## 验证步骤

- [ ] Redis `INFO stats` 中 `keyspace_hits` / (`keyspace_hits` + `keyspace_misses`) > 80%
- [ ] 数据库 QPS 恢复至基线
- [ ] 接口响应时间恢复至基线
- [ ] 无新的 `Cache miss` 突增
- [ ] Redis 集群状态 `cluster_state:ok`

------

## 升级路径

| 条件                       | 升级动作                               |
| :------------------------- | :------------------------------------- |
| Redis 集群节点宕机无法恢复 | 联系 @cache-team / 云厂商工单          |
| 数据库已被打挂             | 立即升级 P1，联系 @dba-oncall + @cto   |
| 需要调整全局缓存架构       | 联系 @架构师（多级缓存、热点探测方案） |
| 疑似缓存穿透攻击           | 联系 @security-oncall                  |

------

## 相关链接

| 链接           | 说明                 |
| :------------- | :------------------- |
| Redis 监控大盘 | Redis 集群监控       |
| DB 监控大盘    | 数据库负载监控       |
| RB-OPS-001     | 微服务上线故障处理   |
| RB-OPS-002     | 数据库连接池耗尽处理 |

------

## 变更记录

| 版本 | 日期       | 修改人    | 变更内容                           |
| :--- | :--------- | :-------- | :--------------------------------- |
| v1.0 | 2026-07-16 | @sre-team | 初稿，覆盖缓存雪崩、击穿、穿透场景 |