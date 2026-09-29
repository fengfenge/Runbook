# RB-OPS-002：数据库连接池耗尽故障处理 Runbook

> **元数据**
> - 文档名称: 数据库连接池耗尽
> - 适用服务: 所有使用 JDBC 连接池的微服务
> - 负责人: @sre-oncall / @dba-oncall
> - 最后更新: 2026-07-16
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P2（部分接口超时）/ P1（服务完全不可用）

---

## 触发条件

- [ ] 监控告警：`dataSource_activeConnections == maxPoolSize` 持续 2 分钟
- [ ] 应用日志出现：`Connection pool is exhausted`、`Cannot get a connection`
- [ ] 应用日志出现：`HikariPool-1 - Thread starvation or clock leap detected`
- [ ] 大量接口响应时间突增，最终超时
- [ ] 线程 Dump 中大量线程卡在 `getConnection()`
- [ ] 数据库侧 `show processlist` 连接数接近 `max_connections`

---

## 前置检查清单（1 分钟）

- [ ] 已连接生产 VPN / 跳板机
- [ ] `kubectl` 上下文已切换至 `prod` 集群
- [ ] 已获取数据库只读账号（用于 `show processlist`）
- [ ] 已通知值班群：`「<service-name> 连接池耗尽，排查中」`
- [ ] 已确认数据库主库状态正常（非数据库本身故障）

---

## 快速诊断

### 步骤 1：确认连接池状态（1 分钟）

```bash
# 进入 Java 应用 Pod
kubectl exec -it <pod-name> -n prod -- /bin/sh

# 方式1：通过 Actuator（如已开启）
curl -s http://localhost:8080/actuator/metrics/jdbc.connections.active
curl -s http://localhost:8080/actuator/metrics/jdbc.connections.max

# 方式2：通过 Arthas（推荐）
java -jar /app/arthas/arthas-boot.jar --attach $(pgrep java)

# 进入 Arthas 后执行：
vmtool --action getInstances --className com.zaxxer.hikari.HikariPool --limit 5
```

**关键指标判断**：

| 指标                | 正常      | 异常  | 含义             |
| :------------------ | :-------- | :---- | :--------------- |
| `activeConnections` | < 70% max | ≈ max | 连接池已满       |
| `idleConnections`   | > 20%     | ≈ 0   | 无空闲连接可用   |
| `pendingThreads`    | 0         | > 0   | 有线程在等待连接 |
| `totalConnections`  | ≈ max     | < max | 实际创建的连接数 |

------

### 步骤 2：查看线程 Dump（2 分钟）

```bash
# 在 Pod 内生成线程 Dump
jstack $(pgrep java) > /tmp/thread-dump.txt

# 统计卡在 getConnection 的线程数
grep -c "getConnection\|HikariDataSource.getConnection" /tmp/thread-dump.txt

# 查看具体堆栈
grep -A 20 "getConnection" /tmp/thread-dump.txt | head -60
```

**常见堆栈模式**：

| 堆栈特征                                                    | 根因               | 处理方向                 |
| :---------------------------------------------------------- | :----------------- | :----------------------- |
| `at java.net.SocketInputStream.socketRead0`                 | 慢 SQL 占用连接    | **步骤 3：慢 SQL 排查**  |
| `at java.lang.Thread.sleep`                                 | 连接泄漏（未关闭） | **步骤 4：连接泄漏排查** |
| `at com.mysql.cj.jdbc.ClientPreparedStatement.executeQuery` | 大查询 / 未分页    | **步骤 3：慢 SQL 排查**  |
| `at redis.clients.jedis.JedisPool.getResource`              | 同时卡 Redis       | **步骤 5：级联故障**     |

------

### 步骤 3：数据库侧排查（2 分钟）

```bash
# 登录 MySQL 主库（只读账号）
mysql -h <db-host> -P 3306 -u readonly -p

# 查看当前连接
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';

# 查看慢查询（如已开启慢日志）
SELECT * FROM mysql.slow_log WHERE start_time > NOW() - INTERVAL 10 MINUTE ORDER BY query_time DESC LIMIT 10;
```

**关键判断**：

| 现象                                       | 根因                              | 处理                      |
| :----------------------------------------- | :-------------------------------- | :------------------------ |
| 大量 `Sleep` 状态连接，Time > 100s         | 连接未释放 / 未设置 `maxLifetime` | 重启应用 + 调整连接池配置 |
| 大量 `Query` 状态，Time 持续增长           | 慢 SQL / 锁等待                   | Kill 慢查询 + SQL 优化    |
| `Threads_connected` 接近 `max_connections` | 数据库侧连接上限                  | 联系 DBA 扩容             |
| 存在 `Waiting for table lock`              | 锁竞争 / 未提交事务               | 查找持有锁的会话并 Kill   |

**紧急 Kill 慢查询**：

```sql
-- 查看执行时间超过 60 秒的查询
SELECT id, user, host, db, command, time, state, info 
FROM INFORMATION_SCHEMA.PROCESSLIST 
WHERE time > 60 AND command != 'Sleep';

-- Kill 指定线程（谨慎操作）
KILL <thread_id>;
```

------

### 步骤 4：连接泄漏排查（2 分钟）



```bash
# 查看应用日志中连接获取/关闭的异常
kubectl logs <pod-name> -n prod --tail=500 | grep -E "Connection|HikariPool|SQLException"

# 检查代码中常见的泄漏模式（通过日志中的 SQL 定位）
```

**常见泄漏原因**：

| 原因                                         | 代码特征                                         | 修复                              |
| :------------------------------------------- | :----------------------------------------------- | :-------------------------------- |
| 连接未在 finally 中关闭                      | `try { conn = ds.getConnection() }` 无 `finally` | 使用 try-with-resources           |
| 流 / ResultSet 未关闭                        | `rs = stmt.executeQuery()` 后未 `rs.close()`     | 统一关闭顺序                      |
| 事务未提交/回滚                              | `setAutoCommit(false)` 后异常退出                | 确保事务边界完整                  |
| 连接池 `maxLifetime` > 数据库 `wait_timeout` | 连接被数据库关闭但池不知道                       | 调整 `maxLifetime < wait_timeout` |

------

### 步骤 5：级联故障排查（1 分钟）

**检查是否因下游故障导致线程占用**：

```bash
# 查看线程 Dump 中是否同时卡在 HTTP / Redis / MQ 调用
grep -c "HttpClient\|RestTemplate\|Feign\|Jedis\|KafkaProducer" /tmp/thread-dump.txt
```

**判断**：如果大量线程卡在下游调用，且持有数据库连接不释放 → **下游故障引发的级联反应**，应先处理下游或启用降级。

------

## 处置措施

### 紧急扩容连接池

```bash
# 通过环境变量或配置中心临时扩容
# 方式1：修改 Deployment 环境变量（需重启）
kubectl patch deployment <service-name> -n prod -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<service-name>",
          "env": [
            {"name": "DB_MAX_POOL_SIZE", "value": "100"},
            {"name": "DB_CONNECTION_TIMEOUT", "value": "5000"}
          ]
        }]
      }
    }
  }
}'

# 触发滚动重启
kubectl rollout restart deployment/<service-name> -n prod
```

------

### 重启应用

```bash
# 当确认是连接泄漏且无法快速修复代码时
kubectl rollout restart deployment/<service-name> -n prod

# 观察重启后连接数
kubectl exec -it <new-pod> -n prod -- jcmd 1 VM.heap_summary
```

> ⚠️ **注意**：重启只是临时止血，必须找到泄漏根因并修复。

------

### 措施 C：Kill 数据库慢查询



```sql
-- 在 MySQL 中执行
-- 1. 找出耗时最长的非 Sleep 连接
SELECT id, time, info FROM INFORMATION_SCHEMA.PROCESSLIST 
WHERE command != 'Sleep' ORDER BY time DESC LIMIT 10;

-- 2. 逐个 Kill（每次 Kill 后观察 1 分钟）
KILL <id1>;
KILL <id2>;
```

------

### 措施 D：启用降级（保护核心链路）



```bash
# 通过配置中心关闭非核心功能的数据库访问
curl -X POST http://config-center/api/switch \
  -d '{"service":"<service-name>","feature":"non-core-db","enabled":"false"}'

# 或调整 API Gateway 限流
curl -X POST http://gateway/api/rate-limit \
  -d '{"service":"<service-name>","qps":100}'
```

------

## 连接池配置检查清单

**事后必须检查的配置项**：



| 配置项                   | 推荐值                    | 说明                            |
| :----------------------- | :------------------------ | :------------------------------ |
| `maximumPoolSize`        | CPU 核数 * 2 + 有效磁盘数 | 通常 10-50                      |
| `minimumIdle`            | = `maximumPoolSize`       | 避免连接创建开销                |
| `connectionTimeout`      | 3000-5000 ms              | 获取连接最大等待时间            |
| `idleTimeout`            | 600000 ms (10分钟)        | 空闲连接回收                    |
| `maxLifetime`            | 1800000 ms (30分钟)       | 必须 < 数据库 `wait_timeout`    |
| `leakDetectionThreshold` | 60000 ms                  | 开发/测试环境开启，用于发现泄漏 |

------

## 验证步骤

- [ ] `activeConnections` < 70% `maximumPoolSize`
- [ ] `pendingThreads` = 0
- [ ] 接口响应时间恢复至基线
- [ ] 无新的 `Connection pool exhausted` 日志
- [ ] 数据库 `Threads_connected` 稳定下降

------

## 升级路径

| 条件                              | 升级动作                        |
| :-------------------------------- | :------------------------------ |
| Kill 慢查询后问题反复             | 联系 @dba-oncall 进行 SQL 优化  |
| 连接泄漏根因无法定位              | 联系 @开发-team 代码审查        |
| 数据库 `max_connections` 已达上限 | 联系 @dba-oncall 扩容数据库规格 |
| 疑似数据库主库性能瓶颈            | 联系 @dba-oncall + @架构师      |

------

## 相关链接

| 链接              | 说明               |
| :---------------- | :----------------- |
| [Grafana DB 监控] | 数据库连接数监控   |
| [Arthas 文档]     | JVM 在线诊断       |
| RB-OPS-001        | 微服务上线故障处理 |
| RB-OPS-003        | Redis 缓存雪崩处理 |

------

## 变更记录

| 版本 | 日期       | 修改人    | 变更内容                             |
| :--- | :--------- | :-------- | :----------------------------------- |
| v1.0 | 2026-07-16 | @sre-team | 初稿，覆盖 HikariCP / Druid 常见场景 |