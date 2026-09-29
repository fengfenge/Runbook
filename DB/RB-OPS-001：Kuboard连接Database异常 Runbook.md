# RB-OPS-001：Kuboard连接Database异常 Runbook

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

- [x] 用户反馈：Web页面无法正常显示

## 前置检查清单

- [x] 已连接进入ecs
- [x] 进入docker部署的kuboard服务
- [x] 已通知值班群：`「<service-name> 系统繁忙故障排查中，@oncall」`



## 快速诊断

### 步骤1：查看web报错

```bash
http://iot-amqp.tian-power.com/sso/auth?access_type=offline&client_id=kuboard-sso&redirect_uri=%2Fcallback&response_type=code&scope=openid+profile+email+groups&state=%2F&connector_id=default
{
  "message": "Failed to connect to the database.",
  "type": "Internal Server Error"
}
```

### 步骤2：查看容器kuboard报错

```bash
# 查看最近33条日志
docker logs --tail 33 kuboard
# 跟踪日志（会全部展示）
docker logs -f kuboard

time="2026-07-30T03:25:10Z" level=error msg="Storage health check failed: create auth request: etcdserver: mvcc: database space exceeded"
{"level":"warn","ts":"2026-07-30T11:25:25.311+0800","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-591df4e6-e4f3-4265-a930-36d55795354c/127.0.0.1:2379","attempt":0,"error":"rpc error: code = ResourceExhausted desc = etcdserver: mvcc: database space exceeded"}
time="2026-07-30T03:25:25Z" level=error msg="Storage health check failed: create auth request: etcdserver: mvcc: database space exceeded"
```

### 步骤3：日志审批

```bash
# etcdctl --write-out=table endpoint status ，# DB SIZE 为 2.1 GB（已超默认配额）。
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------------------------------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX |             ERRORS             |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------------------------------+
| 127.0.0.1:2379 | 59a9c584ea2c3f35 |  3.4.14 |  2.1 GB |      true |      false |         4 |    6081756 |            6081756 |   memberID:6460912315094810421 |
|                |                  |         |         |           |            |           |            |                    |                 alarm:NOSPACE  |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------------------------------+
```

输出中 `alarm:NOSPACE` 明确表明空间已满

- 已在 etcd 所在服务器上，且有 `etcdctl` 可执行权限。
- etcd 的存储空间（默认 2GB）已写满，触发 **空间不足（No Space）** 告警，此时 etcd 会**拒绝所有写入操作**，只能读取

### 步骤4：处理故障

```bash
# 获取当前最新的 revision
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*'
6081501

# # 压缩历史版本（Compaction）etcd 会保留历史修订版本，压缩可释放空间
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 compact 6081501
compacted revision 6081501

# 碎片整理（Defragmentation）压缩后，磁盘空间不会立即归还，需要整理：
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 defrag

# 解除告警
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 alarm disarm
```



### 步骤5：验证

```bash
#确认 DB SIZE 下降，且 ERRORS 列为空 
etcdctl --write-out=table endpoint status 
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 127.0.0.1:2379 | 59a9c584ea2c3f35 |  3.4.14 |  156 kB |      true |      false |         4 |    6081804 |            6081804 |        |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```