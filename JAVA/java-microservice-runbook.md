# Java 微服务故障排查 Runbook

面向角色：云计算运维工程师、SRE、平台运维、值班工程师  
适用范围：运行在 Kubernetes、ECS/VM、容器平台或 PaaS 上的 Java 微服务，包括 Spring Boot、Spring Cloud、Dubbo、gRPC、REST API、消息消费服务、定时任务服务等。  
目标：在告警发生后，快速判断影响范围、定位故障层级、恢复服务、保留证据并推动根因修复。

---

## 1. 快速响应原则

### 1.1 优先级顺序

1. 先恢复业务，再深入根因。
2. 先判断影响面，再处理单点异常。
3. 先排除外部依赖，再深入 JVM 内部。
4. 先保留关键现场，再重启或扩容。
5. 所有变更动作必须可回滚、可追踪、可解释。

### 1.2 值班响应 SLA

| 故障等级 | 典型表现 | 首次响应 | 临时止血 | 根因初判 |
|---|---|---:|---:|---:|
| P0 | 核心链路不可用、大面积 5xx、支付/登录/下单失败 | 5 分钟 | 15 分钟 | 30 分钟 |
| P1 | 单核心服务严重降级、错误率明显升高 | 10 分钟 | 30 分钟 | 60 分钟 |
| P2 | 局部异常、部分租户/区域受影响 | 30 分钟 | 2 小时 | 1 个工作日 |
| P3 | 非核心告警、容量趋势、偶发异常 | 1 个工作日 | 按计划 | 按计划 |

### 1.3 故障处理流程

```text
收到告警
  -> 确认告警真实性
  -> 判断影响范围和故障等级
  -> 建立事件群/记录事件单
  -> 查看近期变更
  -> 按层级排查：流量 -> 网关 -> 服务 -> JVM -> 依赖 -> 基础设施
  -> 采取止血动作
  -> 验证业务恢复
  -> 保留现场和关键证据
  -> 根因分析
  -> 复盘与长期修复
```

---

## 2. 故障信息收集模板

### 2.1 事件基本信息

| 项目 | 内容 |
|---|---|
| 事件编号 | INC-YYYYMMDD-XXX |
| 发现时间 |  |
| 恢复时间 |  |
| 故障等级 | P0 / P1 / P2 / P3 |
| 影响系统 |  |
| 影响接口/任务/消费者 |  |
| 影响租户/地域/集群 |  |
| 业务影响 |  |
| 当前负责人 |  |
| 相关研发 |  |
| 最近变更 | 发布 / 配置 / 扩缩容 / 网络 / 数据库 / 中间件 |

### 2.2 关键证据清单

排查过程中尽量保留以下信息：

- 告警截图或告警详情。
- 监控大盘截图，包括 QPS、错误率、延迟、CPU、内存、GC、线程数、连接数。
- 服务日志中的异常堆栈。
- Pod 或主机事件。
- 最近发布记录、配置变更、镜像版本、Git Commit。
- 依赖服务状态，包括数据库、Redis、MQ、注册中心、配置中心、对象存储、第三方 API。
- JVM 现场，包括 heap dump、thread dump、GC 日志、JFR 文件。

---

## 3. 常用排查入口

### 3.1 Kubernetes 服务

```bash
# 查看命名空间
kubectl get ns

# 查看服务整体状态
kubectl -n <namespace> get deploy,sts,ds,svc,ingress,pod -o wide

# 查看异常 Pod
kubectl -n <namespace> get pod | egrep -v 'Running|Completed'

# 查看 Pod 事件
kubectl -n <namespace> describe pod <pod-name>

# 查看最近事件
kubectl -n <namespace> get events --sort-by='.lastTimestamp'

# 查看 Deployment 发布历史
kubectl -n <namespace> rollout history deploy/<deploy-name>

# 查看当前镜像
kubectl -n <namespace> get deploy <deploy-name> -o jsonpath='{.spec.template.spec.containers[*].image}'

# 查看日志
kubectl -n <namespace> logs <pod-name> -c <container-name> --tail=300
kubectl -n <namespace> logs <pod-name> -c <container-name> --previous --tail=300

# 进入容器
kubectl -n <namespace> exec -it <pod-name> -c <container-name> -- sh
```

### 3.2 ECS/VM 服务

```bash
# 查看服务状态
systemctl status <service-name>
journalctl -u <service-name> -n 300 --no-pager

# 查看进程
ps -ef | grep java
pgrep -fa java

# 查看端口
ss -lntp
ss -antp | grep <port>

# 查看系统资源
top
free -h
df -h
iostat -x 1
vmstat 1
```

### 3.3 Java 进程

```bash
# 查找 Java 进程
jps -lv
pgrep -fa java

# 查看 JVM 参数
jcmd <pid> VM.flags
jcmd <pid> VM.command_line

# 查看 JVM 运行时信息
jcmd <pid> VM.system_properties
jcmd <pid> VM.version

# 查看线程
jstack -l <pid> > jstack-$(date +%F-%H%M%S).log
jcmd <pid> Thread.print -l > thread-$(date +%F-%H%M%S).log

# 查看堆信息
jmap -heap <pid>
jcmd <pid> GC.heap_info

# 生成堆转储，注意磁盘空间和业务影响
jcmd <pid> GC.heap_dump /tmp/heap-$(date +%F-%H%M%S).hprof

# 查看类加载和内存统计
jcmd <pid> GC.class_histogram > class-histogram-$(date +%F-%H%M%S).log
```

---

## 4. 标准排查路径

### 4.1 第一步：确认是否真实故障

检查项：

- 告警是否持续触发，还是瞬时抖动。
- 业务监控是否同步异常。
- 用户投诉、客服反馈、工单是否增加。
- 合成探测、健康检查是否失败。
- 多个可观测系统是否一致，例如 Prometheus、APM、日志平台、网关监控。

常用指标：

| 指标 | 判断方式 |
|---|---|
| QPS | 是否突增、突降、归零 |
| 错误率 | 4xx 与 5xx 分开看 |
| P95/P99 延迟 | 是否超过 SLO |
| Apdex | 是否明显下降 |
| 饱和度 | CPU、内存、线程池、连接池、队列长度 |
| 依赖错误 | DB、Redis、MQ、第三方接口是否异常 |

### 4.2 第二步：确认影响范围

从以下维度切分：

- 单 Pod、单节点、单可用区、单集群，还是全局。
- 单接口、单消费者、单定时任务，还是整个服务。
- 单租户、单用户组、单地域，还是全部用户。
- 只读链路、写链路，还是核心交易链路。
- 内网调用异常，还是公网入口异常。

判断建议：

```text
单 Pod 异常：优先看 Pod 事件、容器重启、节点资源、JVM。
单节点异常：优先看节点资源、内核、磁盘、网络、CNI。
单接口异常：优先看接口日志、下游依赖、慢 SQL、线程池。
全服务异常：优先看发布、配置、注册发现、网关、数据库、中间件。
全局异常：优先看基础设施、云产品、DNS、证书、统一配置、共享依赖。
```

### 4.3 第三步：确认是否近期变更导致

需要检查：

- 最近是否发布新版本。
- 是否修改配置中心配置。
- 是否调整 JVM 参数。
- 是否调整限流、熔断、网关路由。
- 是否修改数据库表结构、索引、权限。
- 是否变更 MQ Topic、Consumer Group、订阅关系。
- 是否扩缩容、节点迁移、证书轮换、DNS 切换。

Kubernetes 常用命令：

```bash
kubectl -n <namespace> rollout history deploy/<deploy-name>
kubectl -n <namespace> describe deploy <deploy-name>
kubectl -n <namespace> get rs -l app=<app-name> -o wide
```

止血建议：

| 变更类型 | 优先止血动作 |
|---|---|
| 应用发布 | 回滚到上一稳定版本 |
| 配置变更 | 回滚配置并刷新实例 |
| 流量切换 | 暂停切流或切回旧集群 |
| 数据库变更 | 停止写入风险链路，联系 DBA |
| 中间件变更 | 恢复订阅、路由、权限或白名单 |

### 4.4 第四步：从入口到依赖逐层排查

```text
用户请求
  -> DNS / CDN / WAF / SLB
  -> Ingress / API Gateway
  -> Service / Pod
  -> Java 应用
  -> JVM 资源
  -> 线程池 / 连接池
  -> DB / Redis / MQ / 第三方 API
  -> 存储 / 网络 / 节点 / 云产品
```

---

## 5. Java 微服务关键指标

### 5.1 应用层指标

| 指标 | 异常表现 | 常见原因 |
|---|---|---|
| HTTP 5xx | 服务端错误升高 | 代码异常、依赖失败、连接池耗尽 |
| HTTP 4xx | 客户端错误升高 | 参数变更、鉴权失败、路由错误 |
| P95/P99 延迟 | 响应变慢 | 慢 SQL、GC、锁竞争、下游慢 |
| QPS | 突增或突降 | 流量峰值、入口故障、调用方异常 |
| 业务失败率 | 业务码失败升高 | 业务规则、库存、账户、权限异常 |
| 限流量 | 被限流请求增加 | 流量过高、阈值过低、实例不足 |
| 熔断量 | 熔断打开 | 下游不稳定或超时配置不合理 |

### 5.2 JVM 指标

| 指标 | 风险信号 | 处理方向 |
|---|---|---|
| Heap 使用率 | 持续高于 85%，Full GC 后不下降 | 内存泄漏或堆太小 |
| Full GC 次数 | 频繁 Full GC | 内存压力、对象晋升、元空间问题 |
| GC 暂停时间 | STW 时间过长 | 调整堆、GC 策略、降低分配速率 |
| Thread Count | 持续增长 | 线程泄漏、阻塞、线程池无界 |
| Deadlock | 检测到死锁 | 分析 jstack 并重启止血 |
| Metaspace | 持续上涨 | 类加载泄漏、动态代理过多 |
| Direct Memory | 堆外内存增长 | Netty、NIO、压缩、缓存问题 |

### 5.3 容器与主机指标

| 指标 | 风险信号 | 常见原因 |
|---|---|---|
| CPU 使用率 | 长时间接近 limit | 流量高、死循环、GC、加密压缩 |
| CPU Throttling | 容器被频繁限速 | CPU limit 过低 |
| Memory Working Set | 接近 limit | 堆、堆外、缓存、内存泄漏 |
| OOMKilled | 容器被杀 | 内存 limit 不足或泄漏 |
| 磁盘使用率 | 超过 85% | 日志膨胀、dump 文件过大 |
| 网络重传 | 明显升高 | 网络质量、连接过多、MTU、CNI |
| 文件句柄 | 接近限制 | 连接泄漏、句柄泄漏 |

---

## 6. 常见故障场景 Runbook

## 6.1 服务不可用或大量 5xx

### 现象

- 网关、APM 或调用方告警 5xx 升高。
- 健康检查失败。
- Pod Ready 数下降。
- 业务接口返回 500、502、503、504。

### 排查步骤

1. 查看入口错误码分布。

```bash
# Kubernetes
kubectl -n <namespace> get pod -l app=<app-name> -o wide
kubectl -n <namespace> logs deploy/<deploy-name> --tail=300
```

2. 判断是应用返回 5xx，还是网关或负载均衡返回 5xx。

| 错误码 | 常见含义 |
|---|---|
| 500 | 应用内部异常 |
| 502 | 网关连接后端失败、后端异常关闭连接 |
| 503 | 无可用实例、服务过载、熔断 |
| 504 | 网关等待后端超时 |

3. 检查 Pod 是否重启、未就绪、镜像拉取失败。

```bash
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> get events --sort-by='.lastTimestamp'
```

4. 查看应用日志异常。

重点搜索：

```text
Exception
ERROR
Timeout
Connection refused
Connection reset
OutOfMemoryError
RejectedExecutionException
CannotGetJdbcConnectionException
RedisConnectionFailureException
```

5. 检查下游依赖是否异常，包括数据库、Redis、MQ、第三方 API。

### 止血动作

- 回滚最近版本。
- 扩容服务实例。
- 降低入口流量或开启限流。
- 临时关闭非核心功能。
- 切换到备用依赖或降级返回兜底数据。
- 重启异常实例，重启前尽量保留日志和 JVM 现场。

### 恢复验证

- 5xx 恢复到基线。
- P95/P99 延迟恢复。
- Ready 实例数量正常。
- 核心业务探测通过。
- 无新的错误日志持续产生。

---

## 6.2 服务响应慢或超时

### 现象

- P95/P99 延迟升高。
- 网关 504 或调用方超时。
- 线程池队列堆积。
- 数据库慢查询增加。

### 排查步骤

1. 先定位慢在哪一层。

| 层级 | 排查方式 |
|---|---|
| 网关层 | 看网关 upstream latency |
| 应用层 | 看接口耗时、APM Trace |
| JVM 层 | 看 GC、线程阻塞、CPU |
| DB 层 | 看慢 SQL、锁等待、连接池 |
| Redis 层 | 看慢命令、大 Key、连接数 |
| MQ 层 | 看消费延迟、积压、重试 |

2. 查看线程栈。

```bash
jcmd <pid> Thread.print -l > thread-$(date +%F-%H%M%S).log
```

重点关注：

- 大量线程处于 `BLOCKED`。
- 大量线程处于 `WAITING` 或 `TIMED_WAITING` 且卡在同一依赖。
- HTTP 线程池被占满。
- 业务线程池队列堆积。
- 大量线程卡在数据库、Redis、HTTP Client、MQ Client。

3. 查看 GC。

```bash
jstat -gcutil <pid> 1000 10
jcmd <pid> GC.heap_info
```

4. 查看数据库慢查询和连接池。

重点检查：

- SQL 是否走索引。
- 是否出现锁等待。
- 连接池 active 是否接近 max。
- 连接获取耗时是否升高。
- 是否有长事务。

### 止血动作

- 扩容应用实例。
- 临时提高线程池或连接池上限，前提是下游能承受。
- 对慢接口限流或降级。
- 回滚导致慢查询的版本。
- 杀掉异常长事务或慢查询，需遵循数据库变更审批。
- 暂停非核心消费任务，保护核心链路。

---

## 6.3 CPU 使用率高

### 现象

- 容器或主机 CPU 接近 100%。
- 接口延迟升高。
- GC 时间增加。
- CPU throttling 明显。

### 排查步骤

1. 找到高 CPU 进程和线程。

```bash
top -H -p <pid>
```

2. 将线程 ID 转为十六进制。

```bash
printf "%x\n" <tid>
```

3. 导出线程栈并查找对应 nid。

```bash
jstack -l <pid> > jstack.log
grep -i "nid=0x<hex-tid>" -A 50 jstack.log
```

4. 判断类型。

| 线程栈特征 | 可能原因 |
|---|---|
| GC Thread 占用高 | GC 压力 |
| JSON 序列化/反序列化 | 大对象或循环处理 |
| 正则表达式 | 复杂正则回溯 |
| 加密/压缩 | CPU 密集型任务 |
| 日志格式化 | 日志量过大 |
| 业务循环 | 死循环或大批量计算 |

5. 检查容器 CPU limit 是否过低。

```bash
kubectl -n <namespace> describe pod <pod-name> | grep -A 5 -i limits
```

### 止血动作

- 扩容实例分摊 CPU。
- 临时提高 CPU limit。
- 关闭或降级 CPU 密集功能。
- 降低日志级别。
- 回滚异常版本。
- 对异常接口限流。

---

## 6.4 内存高、频繁 GC 或 OOM

### 现象

- Heap 持续上涨。
- Full GC 频繁。
- Pod 被 OOMKilled。
- 日志出现 `OutOfMemoryError`。
- 服务短时间反复重启。

### 排查步骤

1. 判断是 JVM 堆内存、堆外内存，还是容器整体内存。

```bash
kubectl -n <namespace> describe pod <pod-name>
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary
```

如果未开启 Native Memory Tracking，需要在启动参数中加入：

```text
-XX:NativeMemoryTracking=summary
```

2. 查看 GC 状态。

```bash
jstat -gcutil <pid> 1000 10
```

3. 导出对象直方图。

```bash
jcmd <pid> GC.class_histogram > class-histogram.log
```

4. 必要时生成 heap dump。

```bash
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

注意事项：

- heap dump 文件可能很大，先确认磁盘空间。
- dump 期间进程可能暂停。
- 生产环境应优先在故障副本或低峰时执行。
- dump 文件可能包含敏感数据，应加密保存并限制访问。

### 常见原因

| 类型 | 表现 |
|---|---|
| 堆内存泄漏 | Full GC 后 Heap 不下降 |
| 缓存无限增长 | Map、Guava Cache、Caffeine 配置不合理 |
| 大对象 | 查询或导出一次加载过多数据 |
| 堆外内存泄漏 | Netty DirectByteBuffer、NIO、压缩库 |
| 线程泄漏 | 线程数量持续增长，栈内存占用增加 |
| Metaspace 泄漏 | 动态类、Groovy、CGLIB、热加载 |

### 止血动作

- 扩容实例或提高内存 limit。
- 临时降低流量。
- 关闭大查询、导出、批处理任务。
- 清理异常缓存或重启实例。
- 回滚引入泄漏的版本。
- 调整 JVM 堆大小与容器 limit 的比例。

### JVM 容器参数建议

```text
-XX:InitialRAMPercentage=50
-XX:MaxRAMPercentage=70
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
```

---

## 6.5 线程池耗尽

### 现象

- 日志出现 `RejectedExecutionException`。
- 接口请求堆积。
- 消费速度下降。
- 线程数达到上限。
- 服务看似存活但不处理请求。

### 排查步骤

1. 查看线程池监控。

重点指标：

- active threads。
- pool size。
- queue size。
- completed task count。
- rejected count。

2. 导出线程栈。

```bash
jcmd <pid> Thread.print -l > thread.log
```

3. 判断线程被什么阻塞。

常见阻塞点：

- 数据库连接获取。
- Redis 请求。
- HTTP 下游调用。
- 锁等待。
- MQ 发送确认。
- 文件 IO 或对象存储上传。

### 止血动作

- 对慢接口限流。
- 降级或熔断慢下游。
- 扩容应用实例。
- 临时调整线程池和队列大小。
- 缩短下游超时时间。
- 关闭异常批任务。

---

## 6.6 数据库连接池耗尽

### 现象

- 日志出现 `Connection is not available`、`CannotGetJdbcConnectionException`。
- 接口大量超时。
- 数据库 active connection 接近上限。
- 慢 SQL 或锁等待增加。

### 排查步骤

1. 查看应用连接池指标。

以 HikariCP 为例：

| 指标 | 含义 |
|---|---|
| hikaricp.connections.active | 正在使用连接数 |
| hikaricp.connections.idle | 空闲连接数 |
| hikaricp.connections.pending | 等待连接线程数 |
| hikaricp.connections.timeout | 获取连接超时次数 |
| hikaricp.connections.max | 最大连接数 |

2. 查看数据库侧连接。

MySQL 示例：

```sql
SHOW PROCESSLIST;
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW ENGINE INNODB STATUS\G
```

PostgreSQL 示例：

```sql
SELECT pid, usename, state, wait_event_type, wait_event, query_start, query
FROM pg_stat_activity
ORDER BY query_start ASC;
```

3. 检查慢 SQL、锁等待和长事务。

### 常见原因

- SQL 变慢导致连接长期占用。
- 连接泄漏，代码未释放连接。
- 事务范围过大。
- 应用实例数增加后总连接数超过数据库承载。
- 连接池 max 配置过大或过小。
- 数据库 CPU、IO、锁竞争异常。

### 止血动作

- 回滚慢 SQL 版本。
- 扩容只读库或切换读流量。
- 临时限流写接口。
- 杀掉确认安全的异常长事务。
- 降低应用实例数或单实例连接池上限，避免压垮数据库。
- 开启缓存或降级非核心查询。

---

## 6.7 Redis 异常

### 现象

- 日志出现 Redis 连接失败、超时。
- 接口延迟升高。
- 缓存命中率下降。
- Redis CPU 或内存高。

### 排查步骤

1. 检查 Redis 连接和延迟。

```bash
redis-cli -h <host> -p <port> PING
redis-cli -h <host> -p <port> INFO
redis-cli -h <host> -p <port> SLOWLOG GET 10
```

2. 查看关键指标。

| 指标 | 风险 |
|---|---|
| used_memory | 内存接近上限 |
| connected_clients | 连接数过高 |
| blocked_clients | 阻塞客户端 |
| instantaneous_ops_per_sec | QPS 异常 |
| evicted_keys | 键被淘汰 |
| keyspace_hits/misses | 命中率异常 |
| slowlog | 慢命令 |

3. 排查大 Key 与热 Key。

```bash
redis-cli --bigkeys -h <host> -p <port>
redis-cli --hotkeys -h <host> -p <port>
```

生产注意：

- `--bigkeys` 会扫描 Key，可能对实例有影响。
- 优先在从库或低峰执行。
- 云 Redis 优先使用云厂商控制台的大 Key、热 Key 分析。

### 止血动作

- 限流依赖 Redis 的接口。
- 对热点 Key 增加本地缓存。
- 拆分大 Key。
- 临时扩容 Redis。
- 调整过期时间，避免缓存雪崩。
- 降级非核心缓存读取。

---

## 6.8 MQ 积压或消费异常

### 现象

- 消息堆积增加。
- 消费延迟升高。
- 消费者频繁重平衡。
- 死信队列增长。
- 日志出现反序列化失败、业务处理异常、提交 offset 失败。

### 排查步骤

1. 判断积压范围。

- 单 Topic 还是多个 Topic。
- 单 Consumer Group 还是多个 Group。
- 单分区还是所有分区。
- 消费者实例是否在线。

2. 查看消费者日志。

重点搜索：

```text
rebalance
deserialize
commit offset
poll timeout
business exception
dead letter
```

3. 检查消费端资源。

- CPU、内存是否不足。
- 线程池是否耗尽。
- 下游数据库或 HTTP 是否慢。
- 单条消息处理是否卡住。
- 是否出现毒丸消息。

### 止血动作

- 扩容消费者实例，但分区数不足时扩容无效。
- 暂停异常 Topic 的消费。
- 跳过或转移毒丸消息，必须记录消息 ID 和 offset。
- 临时提高批量消费数量。
- 降级消费逻辑中的非核心下游。
- 对失败消息进入死信队列，避免阻塞主消费。

### Kafka 常用命令

```bash
kafka-consumer-groups.sh --bootstrap-server <broker> --describe --group <group>
kafka-topics.sh --bootstrap-server <broker> --describe --topic <topic>
```

### RocketMQ 常用命令

```bash
mqadmin consumerProgress -n <namesrv> -g <group>
mqadmin topicStatus -n <namesrv> -t <topic>
mqadmin consumerStatus -n <namesrv> -g <group>
```

---

## 6.9 注册中心或配置中心异常

### 现象

- 服务发现不到实例。
- 调用报 `No provider available`、`NameResolution`、`UnknownHostException`。
- 配置未生效或错误配置大面积生效。
- 实例频繁上下线。

### 排查步骤

1. 检查注册中心健康。

- Nacos、Eureka、Consul、ZooKeeper 是否可用。
- 服务实例是否注册。
- 命名空间、分组、环境是否正确。
- 实例 IP、端口、权重、健康状态是否正确。

2. 检查应用启动日志。

重点关注：

```text
register service
deregister service
config refresh
connection refused
authorization failed
namespace not found
```

3. 检查网络连通性。

```bash
nc -vz <registry-host> <port>
curl -v <registry-health-url>
```

### 止血动作

- 回滚错误配置。
- 临时固定服务地址或切换备用注册中心。
- 重启未正确注册的实例。
- 暂停自动刷新高风险配置。
- 恢复命名空间、权限、白名单。

---

## 6.10 Pod CrashLoopBackOff

### 现象

- Pod 状态为 `CrashLoopBackOff`。
- 重启次数持续增加。
- 服务实例数量不足。

### 排查步骤

```bash
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> logs <pod-name> --previous --tail=300
kubectl -n <namespace> get events --sort-by='.lastTimestamp'
```

常见原因：

| 原因 | 证据 |
|---|---|
| 启动参数错误 | 启动日志报参数异常 |
| 配置缺失 | 配置中心或环境变量缺失 |
| 端口冲突 | bind failed |
| 依赖不可达 | 启动时连接 DB、MQ、注册中心失败 |
| 探针配置过严 | readiness/liveness 失败 |
| OOMKilled | describe pod 显示 OOMKilled |
| 镜像问题 | ImagePullBackOff、启动脚本错误 |

### 止血动作

- 回滚镜像。
- 恢复缺失配置或密钥。
- 放宽探针 initialDelaySeconds、timeoutSeconds、failureThreshold。
- 提高内存 limit。
- 临时关闭启动时强依赖检查。

---

## 6.11 健康检查失败

### 现象

- Pod Running 但 NotReady。
- 流量无法打入服务。
- readiness probe 失败。
- liveness probe 导致重复重启。

### 排查步骤

```bash
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> exec -it <pod-name> -- curl -v http://127.0.0.1:<port>/actuator/health
```

检查项：

- 健康检查路径是否正确。
- 应用端口是否正确。
- Spring Boot actuator 是否启用。
- 健康检查是否依赖 DB、Redis、MQ。
- 启动时间是否超过探针配置。
- 容器内访问正常但 Service 访问异常时，检查 Service selector。

### 止血动作

- 修复探针路径和端口。
- 放宽探针时间。
- 将 readiness 与 liveness 区分。
- 避免 liveness 强依赖外部中间件。
- 对启动慢服务增加 startupProbe。

---

## 6.12 磁盘满或日志暴涨

### 现象

- 节点磁盘使用率高。
- 应用写日志失败。
- Pod 被驱逐。
- Java 服务报 `No space left on device`。

### 排查步骤

```bash
df -h
du -xh --max-depth=1 /var/log | sort -h
find / -xdev -type f -size +1G 2>/dev/null
```

Kubernetes 节点：

```bash
kubectl describe node <node-name>
kubectl get pod -A -o wide | grep <node-name>
```

### 止血动作

- 清理过期日志、临时文件、历史 dump。
- 降低日志级别。
- 修复异常循环报错。
- 扩容磁盘。
- 调整日志轮转策略。
- 迁移高日志量 Pod。

日志轮转建议：

```text
单文件大小上限：100MB - 500MB
保留天数：7 - 30 天
保留总大小：按磁盘容量设置
生产避免 DEBUG 级别长期开启
```

---

## 6.13 网络连接异常

### 现象

- `Connection refused`
- `Connection reset`
- `No route to host`
- `UnknownHostException`
- 请求偶发超时

### 排查步骤

1. DNS 解析。

```bash
nslookup <domain>
dig <domain>
```

2. 端口连通。

```bash
nc -vz <host> <port>
telnet <host> <port>
```

3. HTTP 访问。

```bash
curl -v --connect-timeout 3 http://<host>:<port>/<path>
```

4. 连接状态。

```bash
ss -antp
ss -s
```

5. Kubernetes 网络。

```bash
kubectl -n <namespace> get svc,endpoints
kubectl -n <namespace> describe svc <service-name>
kubectl -n <namespace> get networkpolicy
```

### 常见原因

- DNS 解析异常或缓存过期。
- Service selector 错误，Endpoints 为空。
- 安全组、ACL、NetworkPolicy 拦截。
- 下游端口未监听。
- 连接池连接过期。
- NAT 端口耗尽。
- 跨可用区网络抖动。

### 止血动作

- 切换备用域名或 IP。
- 修复 Service selector。
- 回滚网络策略。
- 扩容 NAT 网关或调整连接复用。
- 降低连接创建频率。
- 缩短连接池 max lifetime。

---

## 7. Spring Boot Actuator 排查

如果服务启用了 Actuator，可使用以下接口辅助排查。

```bash
curl http://<host>:<port>/actuator/health
curl http://<host>:<port>/actuator/metrics
curl http://<host>:<port>/actuator/metrics/jvm.memory.used
curl http://<host>:<port>/actuator/metrics/jvm.gc.pause
curl http://<host>:<port>/actuator/metrics/http.server.requests
curl http://<host>:<port>/actuator/threaddump
curl http://<host>:<port>/actuator/heapdump
```

安全要求：

- 生产环境 Actuator 不应直接暴露公网。
- 敏感端点需要鉴权、IP 白名单或内网访问。
- heapdump、env、configprops 可能包含敏感信息。

---

## 8. 止血动作决策表

| 场景 | 首选动作 | 风险 |
|---|---|---|
| 新版本发布后异常 | 回滚版本 | 可能触发数据兼容问题 |
| 单实例异常 | 摘除或重启实例 | 重启前可能丢失现场 |
| 流量突增 | 扩容、限流 | 下游可能被放大压垮 |
| 下游慢 | 熔断、降级、缩短超时 | 部分功能不可用 |
| 数据库压力高 | 限流写入、关闭慢查询入口 | 影响业务功能 |
| MQ 积压 | 扩容消费者、跳过毒丸消息 | 可能乱序或丢业务上下文 |
| 内存泄漏 | 扩容、重启、回滚 | 重启只能临时恢复 |
| CPU 打满 | 扩容、限流、回滚 | 需避免无限扩容掩盖根因 |
| 配置错误 | 回滚配置 | 配置刷新可能不一致 |

---

## 9. 发布回滚 Runbook

### 9.1 回滚前确认

- 当前异常是否与发布时间吻合。
- 上一版本是否稳定。
- 数据库结构是否兼容上一版本。
- 配置是否需要同步回滚。
- 是否存在灰度、蓝绿、金丝雀流量。

### 9.2 Kubernetes 回滚

```bash
# 查看历史
kubectl -n <namespace> rollout history deploy/<deploy-name>

# 回滚到上一版本
kubectl -n <namespace> rollout undo deploy/<deploy-name>

# 回滚到指定版本
kubectl -n <namespace> rollout undo deploy/<deploy-name> --to-revision=<revision>

# 查看回滚状态
kubectl -n <namespace> rollout status deploy/<deploy-name>
```

### 9.3 回滚后验证

- 新 Pod Ready。
- 错误率下降。
- 延迟恢复。
- 核心接口探测成功。
- 无兼容性异常日志。
- 业务方确认恢复。

---

## 10. 扩缩容 Runbook

### 10.1 适用场景

- 流量突增。
- CPU 或内存资源不足。
- 单实例连接数过高。
- 消费堆积且分区数允许并行消费。

### 10.2 Kubernetes 扩容

```bash
kubectl -n <namespace> scale deploy/<deploy-name> --replicas=<replicas>
kubectl -n <namespace> rollout status deploy/<deploy-name>
kubectl -n <namespace> get pod -l app=<app-name> -o wide
```

### 10.3 扩容风险检查

- 数据库连接总数是否会超过上限。
- Redis、MQ、第三方 API 是否能承受更高并发。
- HPA 是否已经触发，避免手动和自动扩容冲突。
- 是否存在单分区、单锁、单热点 Key 限制。
- 是否会打破许可证、配额、限购限制。

---

## 11. 日志排查规范

### 11.1 日志搜索关键词

```text
ERROR
WARN
Exception
Caused by
Timeout
RejectedExecutionException
OutOfMemoryError
StackOverflowError
Connection refused
Connection reset
Broken pipe
Too many open files
No space left on device
Deadlock
Lock wait timeout
```

### 11.2 日志分析重点

- 第一条异常通常比后续异常更有价值。
- 关注异常发生时间与告警开始时间是否一致。
- 关注 traceId、spanId、requestId、userId、tenantId。
- 区分业务异常和系统异常。
- 区分调用方取消、网关超时、应用内部超时。
- 不要只看当前 Pod，需对比健康 Pod 和异常 Pod。

### 11.3 日志字段建议

生产日志建议包含：

```text
timestamp
level
service
instance
env
traceId
spanId
thread
logger
tenantId
userId
uri
method
status
latency
errorCode
message
```

---

## 12. JVM 现场保留规范

### 12.1 需要保留现场的场景

- OOM。
- 死锁。
- CPU 异常高。
- Full GC 频繁。
- 线程池耗尽。
- 内存疑似泄漏。
- 问题重启后会消失。

### 12.2 推荐采集顺序

```bash
date
jps -lv
jcmd <pid> VM.command_line
jcmd <pid> VM.flags
jcmd <pid> Thread.print -l > thread-1.log
sleep 5
jcmd <pid> Thread.print -l > thread-2.log
sleep 5
jcmd <pid> Thread.print -l > thread-3.log
jcmd <pid> GC.heap_info > heap-info.log
jcmd <pid> GC.class_histogram > class-histogram.log
```

如需 heap dump：

```bash
jcmd <pid> GC.heap_dump /tmp/heap-$(date +%F-%H%M%S).hprof
```

### 12.3 文件归档

建议归档路径：

```text
/data/incident/<incident-id>/<service>/<instance>/
```

建议文件：

```text
thread-1.log
thread-2.log
thread-3.log
heap-info.log
class-histogram.log
gc.log
application.log
pod-describe.txt
events.txt
metrics-screenshot.png
```

---

## 13. 监控告警建议

### 13.1 服务级告警

| 告警项 | 建议阈值 |
|---|---|
| 5xx 错误率 | 连续 5 分钟高于 1% 或高于历史基线 3 倍 |
| P95 延迟 | 连续 5 分钟超过 SLO |
| QPS 突降 | 核心服务低于历史基线 50% |
| 可用实例数 | Ready 实例低于期望副本数 |
| 健康检查失败 | 连续失败 3 次以上 |

### 13.2 JVM 告警

| 告警项 | 建议阈值 |
|---|---|
| Heap 使用率 | 连续 10 分钟高于 85% |
| Full GC 次数 | 5 分钟内超过 3 次 |
| GC 暂停 | P99 超过 1 秒 |
| 线程数 | 超过基线 2 倍 |
| OOM | 任意发生 |

### 13.3 基础设施告警

| 告警项 | 建议阈值 |
|---|---|
| CPU 使用率 | 连续 10 分钟高于 80% |
| CPU Throttling | 连续 5 分钟明显升高 |
| 内存使用率 | 高于 85% |
| 磁盘使用率 | 高于 80% 警告，高于 90% 严重 |
| 文件句柄 | 高于 80% |
| Pod 重启 | 10 分钟内重启超过 3 次 |

---

## 14. 故障升级机制

### 14.1 升级条件

满足任一条件应升级：

- P0 或 P1 故障超过 15 分钟未恢复。
- 影响核心交易、登录、支付、数据写入。
- 故障范围跨多个服务、集群或地域。
- 需要数据库、网络、安全、云厂商协同。
- 出现数据丢失、数据错乱、权限越权风险。
- 运维无法独立完成恢复。

### 14.2 升级对象

| 问题类型 | 升级对象 |
|---|---|
| 应用异常 | 服务研发负责人 |
| 数据库异常 | DBA |
| Redis/MQ 异常 | 中间件负责人 |
| K8s/节点异常 | 容器平台负责人 |
| 网络/DNS/SLB | 网络或云平台负责人 |
| 安全拦截/证书 | 安全团队 |
| 云产品故障 | 云厂商支持 |

### 14.3 升级时必须提供

- 故障开始时间。
- 影响范围。
- 已执行动作。
- 当前监控截图或指标。
- 关键日志。
- 最近变更。
- 需要对方协助的问题。

---

## 15. 复盘模板

### 15.1 时间线

| 时间 | 事件 |
|---|---|
| HH:MM | 告警触发 |
| HH:MM | 值班确认 |
| HH:MM | 建立事件群 |
| HH:MM | 定位到问题层级 |
| HH:MM | 执行止血动作 |
| HH:MM | 业务恢复 |
| HH:MM | 根因确认 |

### 15.2 复盘问题

- 故障根因是什么。
- 为什么没有提前发现。
- 为什么告警没有更早触发。
- 为什么止血耗时较长。
- 是否存在变更流程缺陷。
- 是否存在容量、架构、代码、配置缺陷。
- 是否需要补充监控、预案、自动化。

### 15.3 行动项

| 行动项 | 负责人 | 截止时间 | 状态 |
|---|---|---|---|
|  |  |  | 未开始 |

---

## 16. 生产操作注意事项

### 16.1 禁止事项

- 不确认影响范围就重启全部实例。
- 不保留现场就清理日志或删除 Pod。
- 在高峰期执行高风险扫描或全量 dump。
- 未确认数据兼容性就回滚版本。
- 为了恢复服务直接跳过数据一致性校验。
- 在事件群外执行关键操作。
- 未记录命令和结果。

### 16.2 推荐事项

- 每个关键操作前说明目的和风险。
- 每次只改变一个变量，便于判断效果。
- 操作后观察至少一个完整监控窗口。
- 对核心链路建立合成探测。
- 对高风险操作使用双人确认。
- 故障期间保留统一时间线。

---

## 17. 附录：一页式排查清单

```text
[ ] 告警是否真实，业务是否受影响
[ ] 故障等级是否确认
[ ] 是否建立事件群或事件单
[ ] 是否检查最近发布和配置变更
[ ] 是否确认影响范围：实例 / 节点 / 集群 / 地域 / 租户
[ ] 是否查看入口错误码：4xx / 5xx / 502 / 503 / 504
[ ] 是否查看服务日志第一条异常
[ ] 是否查看 Pod 状态、重启次数、事件
[ ] 是否查看 CPU、内存、磁盘、网络
[ ] 是否查看 JVM：GC、线程、堆、OOM
[ ] 是否查看线程池、连接池、队列
[ ] 是否查看数据库慢查询、锁等待、连接数
[ ] 是否查看 Redis、MQ、注册中心、配置中心
[ ] 是否采取止血动作并记录
[ ] 是否验证业务恢复
[ ] 是否保留现场
[ ] 是否完成复盘和行动项
```

---

## 18. 附录：常用命令速查

### Kubernetes

```bash
kubectl -n <namespace> get pod -o wide
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> logs <pod-name> --tail=300
kubectl -n <namespace> logs <pod-name> --previous --tail=300
kubectl -n <namespace> get events --sort-by='.lastTimestamp'
kubectl -n <namespace> rollout history deploy/<deploy-name>
kubectl -n <namespace> rollout undo deploy/<deploy-name>
kubectl -n <namespace> scale deploy/<deploy-name> --replicas=<replicas>
kubectl -n <namespace> top pod
kubectl top node
```

### Linux

```bash
top
top -H -p <pid>
free -h
df -h
du -xh --max-depth=1 <path> | sort -h
ss -lntp
ss -antp
lsof -p <pid>
ulimit -n
iostat -x 1
vmstat 1
dmesg -T | tail -100
```

### JVM

```bash
jps -lv
jcmd <pid> VM.command_line
jcmd <pid> VM.flags
jcmd <pid> Thread.print -l
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jstat -gcutil <pid> 1000 10
jstack -l <pid>
jmap -heap <pid>
```

### 网络

```bash
curl -v --connect-timeout 3 <url>
nc -vz <host> <port>
telnet <host> <port>
nslookup <domain>
dig <domain>
traceroute <host>
```

---

## 19. 附录：故障沟通模板

### 首次通报

```text
【故障通报】
时间：YYYY-MM-DD HH:MM
等级：P0/P1/P2/P3
影响：影响系统/接口/用户范围
现象：错误率/延迟/不可用表现
当前判断：初步怀疑层级
处理中动作：正在执行的排查或止血动作
下次更新时间：HH:MM
```

### 恢复通报

```text
【故障恢复】
时间：YYYY-MM-DD HH:MM
影响：已恢复的业务范围
恢复动作：回滚/扩容/降级/修复配置等
验证结果：监控指标和业务探测已恢复
后续：继续观察，根因分析和复盘稍后输出
```

### 复盘结论

```text
【故障复盘结论】
根因：
影响范围：
恢复动作：
暴露问题：
长期修复：
行动项：
```

