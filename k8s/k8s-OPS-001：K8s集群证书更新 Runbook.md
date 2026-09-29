# K8s-OPS-001：k8s集群证书更新 Runbook

> **元数据**

> 文档名称: k8s集群证书更新
>
> 适用范围: k8s集群瘫痪
>
> 负责人: @sre-oncall 
>
> 最后更新: 2026-07-24
>
> 下次演练: xxx
>
> 预计耗时: 5-30 分钟
>
> 严重级别:  P1

---

## 触发条件

- [ ] 反馈集群出现问题

---

## 前置检查清单（1 分钟）

- [ ] `kubectl` 上下文已切换至 `prod` 集群
- [ ] 已打开发布监控大盘：
- [ ] 已通知值班群：`service-name; 发布异常，启动故障处理」`

针对 **Kubernetes v1.32.0** 使用 kubeadm 部署的集群，证书更新流程如下

## 完整操作步骤

### 1. 检查证书过期时间

```bash
sudo kubeadm certs check-expiration
```

输出示例：
```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
admin.conf                 Jul 24, 2027 08:15 UTC   364d            no
apiserver                  Jul 24, 2027 08:15 UTC   364d            no
apiserver-etcd-client      Jul 24, 2027 08:15 UTC   364d            no
...
```

---

### 2. 备份现有证书和配置

**生产环境务必先备份**

```bash
sudo cp -rp /etc/kubernetes /etc/kubernetes.bak.$(date +%Y%m%d)
```

---

### 3. 更新所有证书

```bash
sudo kubeadm certs renew all
```

输出示例：
```
[renew] Reading configuration from the cluster...
certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
...
Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.
```

> **多控制平面集群**：需要在**每个 control-plane 节点**上执行此命令。

---

### 4. 重新生成 kubeconfig 文件

```bash
sudo kubeadm init phase kubeconfig all
```

这会更新：
- `/etc/kubernetes/admin.conf`
- `/etc/kubernetes/controller-manager.conf`
- `/etc/kubernetes/scheduler.conf`
- `/etc/kubernetes/super-admin.conf` (v1.28+)

---

### 5. 重启控制平面组件

静态 Pod 由 kubelet 管理，**不能直接用 kubectl 删除**。需通过移出清单文件方式重启：

```bash
# 定义清单目录
MANIFESTS="/etc/kubernetes/manifests"

# 重启 kube-apiserver
sudo mv $MANIFESTS/kube-apiserver.yaml /tmp/
sleep 20
sudo mv /tmp/kube-apiserver.yaml $MANIFESTS/

# 重启 kube-controller-manager
sudo mv $MANIFESTS/kube-controller-manager.yaml /tmp/
sleep 20
sudo mv /tmp/kube-controller-manager.yaml $MANIFESTS/

# 重启 kube-scheduler
sudo mv $MANIFESTS/kube-scheduler.yaml /tmp/
sleep 20
sudo mv /tmp/kube-scheduler.yaml $MANIFESTS/

# 重启 etcd
sudo mv $MANIFESTS/etcd.yaml /tmp/
sleep 20
sudo mv /tmp/etcd.yaml $MANIFESTS/
```

> `sleep 20` 对应 kubelet 的 `fileCheckFrequency` 默认周期，确保 kubelet 检测到清单文件移除并终止旧 Pod。

---

### 6. 更新本地 kubectl 配置

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 7. 重启 kubelet

```bash
sudo systemctl restart kubelet
```

---

### 8. 验证更新结果

```bash
# 检查证书新过期时间
sudo kubeadm certs check-expiration

# 验证集群状态
kubectl get nodes
kubectl get pods -n kube-system

# 验证 API Server 证书
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

---

## 可选：延长证书有效期（不建议）

kubeadm 默认生成 **1 年**有效期的证书。如需延长（如 10 年），需在 **初始化集群时** 指定：

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
certificates:
  duration: 87600h  # 10年
```

对于已运行集群，`kubeadm certs renew` 会基于现有证书属性续期，**无法直接修改有效期**。如需更长期限，可考虑：
1. 使用外部 CA 签发长期证书
2. 定期（每年）执行 `kubeadm upgrade` 自动续期 

---

## 关键注意事项

| 事项                 | 说明                                                        |
| -------------------- | ----------------------------------------------------------- |
| **备份**             | 更新前必须备份 `/etc/kubernetes` 目录                       |
| **多 master**        | 每个控制平面节点都要执行 `certs renew`                      |
| **kubelet 自动轮换** | kubelet 客户端证书默认自动更新，位于 `/var/lib/kubelet/pki` |
| **服务中断**         | 重启 apiserver 期间控制平面短暂不可用                       |
| **CA 证书**          | kubeadm 不支持直接轮换 CA 证书，CA 默认 10 年有效期         |

---

如果集群已经**证书过期导致无法连接**，需要特殊处理（手动生成证书）