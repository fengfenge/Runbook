# RB-OPS-001：MongoDB内存限制故障处理 Runbook

> **Metadata**
>
> - 环境: 预发布uat
>
> - 服务名称: `MongoDB-uat`
> - 告警名称: `系统繁忙`
> - 负责人: @sre-oncall
> - 最后更新: 2026-07-08
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 用户反馈：Web/App 页面显示"系统繁忙，请稍后重试"

## 前置检查清单

- [x] 已连接uat的jumpserver
- [x] `kubectl` 上下文已切换至 `prod` 集群或者登陆对应环境的`kuboard`
- [x] 已打开监控大盘：http://192.168.13.131:3000/
- [x] 已通知值班群：`「<service-name> 系统繁忙故障排查中，@oncall」`



## 快速诊断

### 步骤 1：确认故障范围

```bash
# 查看服务 Pod 状态
kubectl get pods -n general -l app=general-device-public

# 查看服务整体可用性
kubectl get svc general-device-public -n general

NAME  		            				 READY   STATUS    RESTARTS   AGE
general-device-public-57f8dc4b4f-nldcw     1/1     Running   0          2d
```

**判断**：

- `STATUS` 非 `Running` → 跳至 **步骤 3：Pod 级故障**
- `READY` 非 `1/1` → 跳至 **步骤 3：Pod 级故障**
- 全部正常 → 继续 **步骤 2**

### 步骤 2：Pod 级故障排查

```bash
# 查看 Pod 事件
kubectl describe pod general-device-public-57f8dc4b4f-nldcw  -n general | tail -40

# 查看 Pod 日志（最近 200 行 ERROR）
kubectl logs general-device-public-57f8dc4b4f-nldcw  -n general --tail=200 | grep -E "ERROR|FATAL|Exception"

Caused by: com.mongodb.MongoQueryException: Query failed with error code 96 with name 'OperationFailed' and error message 'Executor error during find command :: caused by :: Sort operation used more than the maximum 33554432 bytes of RAM. Add an index, or specify a smaller limit.' on server 192.168.13.77:27017


# 或查看上一个容器日志（崩溃重启时）
kubectl logs general-device-public-57f8dc4b4f-nldcw  -n general --previous --tail=100
```

查到日志关键错误：

```bash
systemLog:
  destination: file
  logAppend: true
  path: /opt/logs/mongodb/mongod.log
storage:
  dbPath: /opt/data/mongodb
  journal:
    enabled: true
processManagement:
  fork: true
  pidFilePath: /opt/data/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo
replication:
  replSetName: "rs1"
  oplogSizeMB: 1024
security:
  authorization: disabled
  keyFile: /opt/server/mongodb/bin/db.key
net:
  port: 27017
  bindIp: 0.0.0.0
 
# 内存调整到1G
setParameter:
  internalQueryExecMaxBlockingSortBytes: 1073741824
  #阿里云mongodb副本集
  internalQueryExecMaxBlockingSortBytes
```



## 验证步骤

- `kubectl get pods -n prod -l app=<service-name>` → 全部 Running & Ready
- Grafana 错误率 < 0.1%，P99 延迟 < 500ms
- 业务核心接口验证通过（curl / 自动化测试）
- 用户侧"系统繁忙"提示消失

## 升级路径

| 条件                    | 升级动作                             |
| :---------------------- | :----------------------------------- |
| 5 分钟内未定位根因      | 联系 @sre-lead                       |
| 扩容/重启后仍无改善     | 联系 @sre-lead + @架构师             |
| 涉及数据不一致 / 丢失   | 立即升级 P1，联系 @cto + @dba-oncall |
| 疑似安全攻击（CC/DDoS） | 联系 @security-oncall                |
| 需要全链路压测验证      | 联系 @性能测试团队                   |

## 相关链接

| 链接                                              | 说明                   |
| :------------------------------------------------ | :--------------------- |
| [Grafana 监控大盘]                                | 服务核心指标           |
| [SkyWalking 链路追踪]                             | 分布式调用链           |
| [Kibana 日志查询]                                 | 应用日志检索           |
| [Arthas 在线诊断](https://arthas.aliyun.com/doc/) | JVM 实时诊断工具       |
| SOP-OPS-001                                       | 微服务上线流程         |
| RB-OPS-011                                        | MySQL 故障处理 Runbook |
| RB-OPS-012                                        | Redis 故障处理 Runbook |

## 变更记录

| 版本 | 日期       | 修改人    | 变更内容 |
| :--- | :--------- | :-------- | :------- |
| v1.0 | 2026-07-08 | @sre-team | 初稿，   |