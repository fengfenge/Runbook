# RB-DEPLOY-001：微服务上线故障处理 Runbook

> **元数据**

> 文档名称: 微服务上线故障处理
>
> 适用范围: 灰度发布 / 全量发布过程中的异常
>
> 负责人: @sre-oncall / @release-engineer
>
> 最后更新: 2026-07-16
>
> 下次演练: xxx
>
> 预计耗时: 5-30 分钟
>
> 严重级别: P2（灰度异常）/ P1（全量回滚）

---

## 触发条件

- [ ] 灰度发布后错误率飙升
- [ ] 全量发布后 P99 延迟突增
- [ ] 发布过程中 Pod 持续 CrashLoopBackOff
- [ ] 业务验收不通过
- [ ] 监控告警：`deployment_&lt;name&gt;_available_replicas &lt; desired_replicas`
- [ ] 用户反馈新版本功能异常

---

## 前置检查清单（1 分钟）

- [ ] 确认当前发布批次（灰度 5% / 20% / 50% / 全量）
- [ ] 确认上一稳定版本号（用于回滚）
- [ ] `kubectl` 上下文已切换至 `prod` 集群
- [ ] 已打开发布监控大盘：
- [ ] 已通知值班群：`service-name; 发布异常，启动故障处理」`

---

## 快速诊断

### 步骤 1：确认发布状态（30 秒）

```bash
# 查看 Deployment 滚动更新状态
kubectl rollout status deployment/service-name -n prod --timeout=30s

# 查看 ReplicaSet 历史版本（用于回滚）
kubectl get rs -n prod -l app=<service-name>
deployment "<service-name>" successfully rolled out

# 查看资源使用状态
kubectl top pods -n prod -l app=<service-name>
```

**异常判断**：

- 输出 `progress deadline exceeded` → **步骤 2：Pod 启动失败**
- 输出 `waiting for rollout to finish` → **步骤 3：滚动更新卡住**

------

### 步骤 2：Pod 启动失败诊断（2 分钟）

```bash
# 查看新 Pod 状态
kubectl get pods -n prod -l app=<service-name> --sort-by=.metadata.creationTimestamp

# 查看最新 Pod 的事件和日志
kubectl describe pod <newest-pod-name> -n prod | tail -40
kubectl logs <newest-pod-name> -n prod --previous 2>/dev/null || kubectl logs <newest-pod-name> -n prod

```

**常见启动失败原因对照表**：

| 现象                                                   | 根因                     | 处置                      |
| :----------------------------------------------------- | :----------------------- | :------------------------ |
| `CrashLoopBackOff` + 应用日志有 `NullPointerException` | 新版本代码 Bug           | **立即回滚**              |
| `CrashLoopBackOff` + `ConfigMap not found`             | 配置未同步               | 检查 ConfigMap 是否创建   |
| `ImagePullBackOff`                                     | 镜像不存在 / Harbor 故障 | 检查镜像 Tag、重新触发 CI |
| `OOMKilled`                                            | 新功能内存需求增加       | 临时扩容内存 limit        |
| `CreateContainerConfigError`                           | Secret / 环境变量缺失    | 检查 Secret 配置          |

------

### 步骤 3：滚动更新卡住诊断（2 分钟）

```bash
# 查看 Deployment 详情
kubectl describe deployment <service-name> -n prod | grep -A 10 "Events"

# 查看新 Pod 为何未 Ready
kubectl get pods -n prod -l app=<service-name> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].message}{"\n"}{end}'
```

**常见原因**：

- **Readiness Probe 失败** → 新 Pod 健康检查接口异常
- **资源不足** → 节点 CPU / 内存不够，Pod 无法调度
- **PVC 挂载失败** → 存储类问题

------

### 步骤 4：灰度指标异常诊断（3 分钟）

```bash
# 对比新旧版本 Pod 的指标
# 1. 列出所有 Pod 及版本标签
kubectl get pods -n prod -l app=<service-name> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# 2. 分别查看新旧版本日志
kubectl logs -l app=<service-name> -n prod --tail=100 | grep -E "ERROR|Exception"

# 3. 查看特定新版本 Pod 日志
kubectl logs <new-pod-name> -n prod --tail=200
```

**关键判断**：

- 仅新版本 Pod 报错 → **新版本 Bug，立即回滚**
- 新旧版本都报错 → **环境/依赖问题，非发布本身**
- 新版本延迟高但无错 → **性能退化，评估是否回滚**

### 步骤 5：检查下游依赖

```bash
# 查看服务依赖的ConfigMap
kubectl get configmap <service-name>-config -n prod -o yaml

# 检查下游服务状态
kubectl get pods -n prod -l app=<downstream-service>
```

**常见问题**:

- 下游服务故障 → 按下游服务Runbook处理，同时考虑启用降级开关
- 配置错误 → 回滚ConfigMap: `kubectl rollout undo configmap <name>`

------

## 处置措施

### 措施 A：立即回滚（最高优先级）

**触发条件**：启动失败、错误率 > 1%、业务验收不通过

```bash
# 回滚到上一个稳定版本
kubectl rollout undo deployment/<service-name> -n prod

# 确认回滚进度
kubectl rollout status deployment/<service-name> -n prod

# 确认旧版本 Pod 全部 Running
kubectl get pods -n prod -l app=<service-name>
```

**回滚后验证**：

- [ ] 错误率恢复至基线
- [ ] P99 延迟恢复至基线
- [ ] 业务核心接口验证通过

------

### 措施 B：暂停发布（灰度异常时）

```bash
# 暂停 Deployment 滚动更新
kubectl rollout pause deployment/<service-name> -n prod

# 恢复发布（确认修复后）
kubectl rollout resume deployment/<service-name> -n prod
```

------

### 措施 C：扩容缓解（资源不足时）

```bash
# 水平扩容
kubectl scale deployment/<service-name> --replicas=6 -n prod

# 或垂直扩容（修改deployment的resources）
kubectl patch deployment <service-name> -n prod -p '{"spec":{"template":{"spec":{"containers":[{"name":"<service-name>","resources":{"limits":{"cpu":"1000m","memory":"1Gi"}}}]}}}}'

# 临时扩容节点或调整资源
kubectl patch deployment <service-name> -n prod -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "<service-name>",
          "resources": {
            "limits": {"cpu": "2000m", "memory": "4Gi"}
          }
        }]
      }
    }
  }
}'


#扩容后观察:
kubectl get pods -n prod -l app=<service-name> -w
等待新Pod `STATUS=Running` 且 `READY=1/1`，观察2分钟指标。
```

------

### 措施 D：修复配置后重新发布

```bash
# 更新 ConfigMap / Secret
kubectl apply -f <fixed-config.yaml> -n prod

# 重启 Deployment 使配置生效
kubectl rollout restart deployment/<service-name> -n prod
```

------

## 发布决策矩阵

| 场景                     | 错误率 | P99 延迟   | 决策                     |
| :----------------------- | :----- | :--------- | :----------------------- |
| 灰度 5% 异常             | > 1%   | > 基线 50% | **暂停发布，排查修复**   |
| 灰度 20% 异常            | > 1%   | > 基线 50% | **立即回滚**             |
| 全量发布后异常           | > 0.5% | > 基线 30% | **立即回滚**             |
| 仅日志有 ERROR，指标正常 | < 0.1% | 正常       | **观察，记录问题单**     |
| 业务验收不通过           | -      | -          | **回滚，修复后重新发布** |

------

## 验证步骤

- [ ] `kubectl get pods -n prod -l app=<service-name>` → 全部 Running & Ready
- [ ] 错误率 < 0.1%，P99 延迟 < 基线 + 20%
- [ ] 无新 ERROR 日志
- [ ] 业务核心场景验收通过
- [ ] 用户侧无异常反馈

------

## 升级路径

| 条件                  | 升级动作                                       |
| :-------------------- | :--------------------------------------------- |
| 回滚后问题仍存在      | 联系 @sre-lead + @架构师（可能是依赖服务故障） |
| 涉及数据迁移异常      | 联系 @dba-oncall + @cto                        |
| 疑似 CI/CD 流水线 Bug | 联系 @devops-team                              |

------

## 相关链接

| 链接            | 说明                 |
| :-------------- | :------------------- |
| Grafana监控大盘 | 发布过程监控         |
| Harbor 镜像仓库 | 镜像版本检查         |
| SOP-OPS-001     | 微服务上线标准流程   |
| RB-OPS-002      | 数据库连接池耗尽处理 |
| RB-OPS-003      | Redis 缓存雪崩处理   |

------

## 变更记录

| 版本 | 日期       | 修改人    | 变更内容                   |
| :--- | :--------- | :-------- | :------------------------- |
| v1.0 | 2026-07-16 | @sre-team | 初稿，覆盖发布过程常见故障 |