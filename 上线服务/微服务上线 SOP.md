# 微服务上线 SOP（标准操作流程）

## 步骤

### 1 上线申请

**执行人**: 开发负责人
**交付物**: 上线申请单

- 填写《服务上线申请》表
- 架构设计文档（依赖关系）
- 测试报告
- 回滚脚本（数据库回滚脚本、配置回滚）

### 2 评审

**执行人**: 工程师

- 评审上线需求
- 确认测试通过
- 确认服务依赖健康（下游服务、数据库）
- 检查日志规范（结构化日志）
- 检查监控覆盖
- 确认CI/CD流水线配置正确

### 3 审批

**执行人**: 团队长

- 确认上线时间（避免业务高峰）
- 生产代码分支MR和tag
- 上线前审批通过

### 4 执行

**执行人**: 运维

- 检查上线申请
- 手动执行
- 检查服务状态
- 检查运行日志
- 提交sql脚本和配置结果，和服务运行清单

**检查命令**:

```bash
# 查看Pod状态
kubectl get pods -n prod -l app=<service-name> -w

# 查看P99延迟
curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job=\"<service-name>\"}[5m]))"
```

### 5 验收检查

**执行人**: 负责人

- 业务场景端到端验证
- 功能验证

### 6 回滚

**触发条件**: 发布后指标异常，或验收不通过

```bash
# Kubernetes 回滚，如果web控制管理可使用。
kubectl rollout undo deployment/<service-name> -n prod

# 指定版本回滚
kubectl rollout undo deployment/<service-name> -n prod --to-revision=<revision-id>

# 数据库回滚（如Schema变更）
# 执行预先准备的回滚脚本
mysql -h <host> -u <user> -p < rollback_v1.2_to_v1.1.sql
```

**回滚后检查**:

- 确认旧版本Pod全部Running
- 通知验证

### 7 紧急修复

**执行人**:  开发

- hotfix分支紧急修复bug
- 合预发布验证通过
- 生产更新
- 执行3~5步骤

### 8 复盘归档

**执行人**: SRE + 开发

- 复盘文档
- 服务故障恢复Runbook
- 3个工作日内完成





