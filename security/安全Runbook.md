# 安全SOP

> **Metadata**
>
> - 环境: 生产
>
> - 服务名称: `阿里云ECS k8s ack worker节点`
> - 告警名称: `ack组件pod大量cpu告警`
> - 负责人: @sre-oncall
> - 最后更新: 2026-07-10
> - 下次演练: xxx
> - 预计耗时: 24 小时
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 短信提示：【阿里云】尊敬的阿里云用户Tia*wer:云盾云安全中心检测到您的服务器出现了紧急安全事件:
  蠕虫病毒命令，建议您立即登录云安全中心控制台-安全告警处理。

## 前置检查清单

- [x] 登录阿里云控制台
- [x] 安全告警：云安全中心控制台 - 检测响应 - 告警
- [x] 已打开监控大盘：http://47.115.141.138:8080/
- [x] 通知安全组：`排查处理，@secure`



## 快速诊断

### 1 发现安全告警

- 在安全中心控制台中 -》检测响应 
- 发现进程异常行为、和代码执行攻击
- 详情：
  - 影响资产和更多信息
- 处理：
  - 处理方式
    - 结束进程，付费处理
    - 加白名单，慎用
    - 忽略，相同告警再次发生，再次告警
    - 已手工处理，手动排查处理
    - 停止容器，付费处理，慎用，会影响业务正常使用



### 2 分析

#### 2.1 云安全中心帮助文档

```http
# 云安全中心帮助文档
https://help.aliyun.com/zh/security-center/product-overview/what-is-security-center?spm=5176.2020520154.console-base_help.dexternal.f36bIdrYIdrYVO
```

#### 2.2 详情

```bash
数据来源
进程启动触发检测
告警原因
该命令常被蠕虫调用，高度怀疑为蠕虫行为。
用户名
root
命令行
mkdir -p /dev/shm/.e_399672_
进程路径
/bin/mkdir
进程ID
3682915
父进程命令行
/bin/sh -s
父进程文件路径
/bin/dash
父进程ID
3682559
进程链
-[2570]  /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id 0d25981d71983378c8c092c0986ad049bfec8dc73163b10a7abf5ca9c155bc3b -address /run/containerd/containerd.sock

    -[3682549]  runc --root /run/containerd/runc/k8s.io --log /run/containerd/io.containerd.runtime.v2.task/k8s.io/d86483389194e49a4b510e8e3cd1b190b6ae1eea71c4002c4083b1dcac465cd9/log.json --log-format json --systemd-cgroup exec --process /tmp/runc-process2987300422 --detach --pid-file /run/containerd/io.containerd.runtime.v2.task/k8s.io/d86483389194e49a4b510e8e3cd1b190b6ae1eea71c4002c4083b1dcac465cd9/62a33859eaa569d35cd66c170c540246bdf1c2854a50e0afdbad46899ae83127.pid d86483389194e49a4b510e8e3cd1b190b6ae1eea71c4002c4083b1dcac465cd9

        -[3682557]  runc init

            -[3682558]  runc init

                -[3682559]  /bin/sh -s

K8s命名空间
kube-system
K8s节点名称
cn-shenzhen.172.16.0.117
K8s Pod
node-local-dns-rgwt4
容器名
node-cache
容器ID
d86483389194e49a4b510e8e3cd1b190b6ae1eea71c4002c4083b1dcac465cd9
镜像ID
registry-cn-shenzhen-vpc.ack.aliyuncs.com/acs/k8s-dns-node-cache@sha256:b06f9d1047b35d4c2349c708516da1888863dfdca11602669e2557cf45933696
镜像名
registry-cn-shenzhen-vpc.ack.aliyuncs.com/acs/k8s-dns-node-cache:v1.22.28.1-5f96b759-aliyun
容器hostname
iZwz9d6709cwgh0rul1ptkZ
容器视角进程路径
/proc/3378/root/bin/mkdir
```

### 3 防护

#### 3.1 ECS专属免费安全防护权益

领取流程：

1. 访问[ECS控制台-实例](https://ecs.console.aliyun.com/server/region)。

2. 选择目标ECS实例，单击**安全防护**页签。

3. 在右侧权益引导页面单击**免费领取**，按提示完成权益领取。

   领取成功后，系统将为您开通[云安全中心](https://yundun.console.aliyun.com/?p=sas)的按量付费实例，并启用**主机安全防护**、**云安全态势管理**及**漏洞修复**功能。默认会对您的ECS实例执行一次扫描，同时[设置并执行周期性自动检查策略](https://help.aliyun.com/document_detail/2653744.html#section-88o-7iq-su4)，以便及时发现和修复ECS实例的配置风险。

   领取后，控制台**免费安全权益**区域显示各功能的免费使用额度：**漏洞修复**20次、**云安全态势管理**1000次、**2核主机病毒防护**90天。

4. 按照系统提示为ECS实例绑定防护版本。更多信息，请参见[管理服务器的防护版本](https://help.aliyun.com/document_detail/2779045.html#e1d00b9fcbyyh)。

   在**安全防护能力**面板的**主机安全防护**区域，状态显示为**未防护**，提示"当前机器病毒防护未生效，请立即绑定"，单击绑定入口为 ECS 实例开启病毒防护



#### 3.2 客户端自保护

客户端自保护启动后，将主动拦截恶意的卸载行为，保障云安全中心防御机制的稳定运转，有效拦截黑客的入侵，防止挖矿、勒索病毒等恶意病毒的扩散。（注意：当前关闭后，自保护机制将在半小时内，自动完成关闭功能）



#### 3.3 漏洞修复

ecs主机实例 - 安全防护

##### 3.3.1 风险

由于云安全中心漏洞修复在测试中无法覆盖所有系统环境，进行漏洞补丁修复行为仍存在一定风险。为了防止出现不可预料的后果，建议您先通过控制台手动创建快照并自行搭建环境充分测试修复方案。



##### 3.3.2 修复决策方案：

当资产中检测出多个漏洞且无法确认优先修复顺序时，可在**漏洞管理**页面，打开**仅显示真实风险漏洞**开关，过滤出修复优先级较高的漏洞。

云安全中心的真实风险漏洞模型依据阿里云漏洞脆弱性评分系统、时间因子、实际环境因子和资产重要性因子对漏洞进行评估，结合实际攻防场景下漏洞是否可被利用（PoC、EXP）及其危害严重性，帮助自动过滤出存在真实安全风险的漏洞。开启该功能可以帮助企业提高可被黑客利用的风险漏洞的修复效率。

对于不同类型的漏洞，云安全中心建议优先修复**待修复应急漏洞**和**Web-CMS漏洞**，这两类漏洞均为阿里云安全工程师确认的高危漏洞。接着再修复应用漏洞、Windows系统漏洞和Linux软件漏洞。

需要根据实际业务情况、服务器的使用情况以及漏洞修复可能造成的影响来判定漏洞是否需要优先修复。



##### 3.3.3 修复流程

为确保漏洞修复过程中目标服务器正常运行，降低异常风险，建议按照以下流程进行修复：

1. 扫描漏洞。

   1. 登录[云安全中心控制台](https://yundun.console.aliyun.com/?p=sas)。在左侧导航栏，选择风险治理 > 漏洞管理。在控制台左上角，选择需防护资产所在的区域：**中国内地**或**非中国内地**。
   2. 在**漏洞管理**页面右上角，单击**漏洞管理设置**。
   3. 在**漏洞管理设置**面板检查漏洞管理设置，确保扫描范围能够覆盖所有服务器的各类漏洞。具体操作，请参见[漏洞管理设置](https://help.aliyun.com/zh/security-center/user-guide/scan-for-vulnerabilities#task-2461190)。
   4. 返回**漏洞管理**页面，单击**一键扫描**。

   检查当前账号下所有服务器的漏洞状态，确保所检测的漏洞信息是即时的。

2. 修复前测试。

   **说明**

   在修复漏洞前，修复人员应在测试环境中部署待修复漏洞的相关补丁，从兼容性和安全性方面进行测试，并在测试完成后编写漏洞修复测试报告。漏洞修复测试报告应包含漏洞修复情况、漏洞修复的时长、补丁本身的兼容性、漏洞修复可能造成的影响。

3. 备份服务器数据。

   **说明**

   为了避免出现不可预料的后果，在正式开始漏洞修复前，修复人员应使用备份恢复系统对待修复漏洞的服务器数据进行备份。例如，使用ECS的快照功能备份目标ECS实例的数据。修复Linux软件漏洞可以使用自动创建快照并修复功能，应急漏洞、应用漏洞需要前往[ECS管理控制台](https://ecs.console.aliyun.com/)创建快照。建议在导出存在漏洞的ECS服务器清单后，使用自动快照功能备份数据。更多信息，请参见[自动快照策略](https://help.aliyun.com/zh/ecs/user-guide/automatically-create-snapshots/#concept-1443642)。

4. 修复漏洞。

   **说明**

   在目标服务器部署修复漏洞的相关补丁及执行修复操作时，应至少有两名修复人员在场，一人负责操作，另一人负责记录，防止出现误操作的情况。

5. 修复后验证。

   **说明**

   修复人员验证目标服务器系统上的漏洞是否已被修复，确保漏洞已修复且目标服务器没有出现任何异常情况。



##### 3.3.4 应急漏洞、应用漏洞

云安全中心只支持检测应急漏洞、应用漏洞并提供修复建议，不支持一键修复。需要根据漏洞详情中提供的修复建议，登录受影响服务器手动修复漏洞。

**说明**

- 由于云安全中心漏洞修复在测试中无法覆盖所有系统环境，漏洞补丁修复行为仍存在一定风险。为了防止出现不可预料的后果，建议先通过[ECS管理控制台](https://ecs.console.aliyun.com/)创建快照并自行搭建环境充分测试修复方案。检测出应急漏洞或应用漏洞的云服务器ECS可以在ECS控制台创建快照进行数据备份。推荐在导出所有存在漏洞的ECS服务器列表后，使用自动快照策略创建快照。更多信息，请参见[自动快照策略](https://help.aliyun.com/zh/ecs/user-guide/automatically-create-snapshots/#concept-1443642)。
- 对于部分因业务影响或未发布安全版本而不能修复的漏洞，建议根据官方提供的临时缓解方法防御攻击。
- 对于业务无影响并且有安全版本的漏洞，建议将软件升级到安全版本。

##### 3.3.5 Linux软件漏洞、Windows系统漏洞

云安全中心支持自动检测和一键修复Linux软件漏洞、Windows系统漏洞，建议在[云安全中心控制台](https://yundun.console.aliyun.com/?p=sas)的漏洞详情页，处理对应漏洞。关于修复漏洞的更多信息，请参见[修复漏洞](https://help.aliyun.com/zh/security-center/user-guide/view-and-handle-vulnerabilities#task-2239619)。

资产中存在多个Linux软件漏洞时，可使用批量修复功能。仅Linux软件漏洞支持批量修复功能。批量修复功能会自动识别选择的漏洞公告对应的资产，并修复这些资产中所选择的漏洞。以下步骤介绍批量修复Linux软件漏洞的具体操作。

1. 登录[云安全中心控制台](https://yundun.console.aliyun.com/?p=sas)。在左侧导航栏，选择**风险治理 > 漏洞管理。在控制台左上角，选择需防护资产所在的区域：**中国内地**或**非中国内地。

2. 在**漏洞管理**页面的**Linux软件漏洞**页签，选中需要批量修复的漏洞并单击**批量修复**。

   **说明**

   批量修复漏洞时，基于性能考虑建议一次修复的漏洞个数不超过100个。需要修复的漏洞超过100个时，可以分次创建快照并进行修复。

3. 在**批量修复**对话框，查看云安全中心识别出的需要修复漏洞的资产列表，选择**自动创建快照并修复**或**不建立快照备份直接修复**，并单击**立即修复**。

如果批量修复漏洞失败，请检查服务器网络连接是否正常、磁盘空间是否已占满。具体操作，请参见[Linux软件漏洞、Windows系统漏洞修复失败，是什么原因？](https://help.aliyun.com/zh/security-center/user-guide/view-and-handle-vulnerabilities#p-1r7-i6p-sah)。

##### 3.3.6 Web-CMS漏洞

云安全中心支持检测并一键修复Web-CMS漏洞。Web-CMS漏洞检测功能可监控网站目录并识别通用建站软件中存在的漏洞。修复Web-CMS漏洞的操作和Linux软件漏洞类似。具体操作，请参见[修复Linux软件漏洞](https://help.aliyun.com/zh/security-center/user-guide/view-and-handle-vulnerabilities#task-2239619)。



##### 3.3.7 生产环境

漏洞修复操作建议

1. **修复前**
   - **资产确认：** 核实服务器资产，确认漏洞相关的软件版本确实存在。
   - **风险评估：** 评估业务影响，判定漏洞修复的紧急性和必要性，并非所有漏洞都需立即修复。
   - **充分测试：** 在测试环境部署补丁，全面验证兼容性与安全性，并输出详细的测试报告。
   - **数据备份：** 对服务器进行完整备份（如创建ECS快照），确保操作失误后可快速回滚。
   - **选择时机：** 选择业务低峰期进行操作，最小化对业务的影响。
2. **修复中**
   - **双人操作：** 需至少两名专业人员在场，一人操作、一人复核记录，防止误操作。
   - **逐项修复：** 严格按照预定的漏洞列表和修复方案，逐一进行修复。
3. **修复后**
   - **结果验证：** 确认漏洞已成功修复，且系统功能和业务应用均运行正常。
   - **文档归档：** 撰写并归档最终的漏洞修复报告，记录完整操作过程。



#### 3.4 云安全中心

```bash
https://help.aliyun.com/zh/security-center/product-overview/what-is-security-center?spm=5176.2020520154.console-base_help.dexternal.49822b72D9UDYt
```



#### 3.5 云安全态势管理

密码登录

秘钥对登录

端口22 

堡垒机

审计日志

##### 3.5.1 秘钥对登录

实例内手动绑定，无需重启

```bash
# 默认保存在/root/.ssh/目录下
ssh-keygen -t rsa -b 2048

生成id_rsa和id_rsa.pub
```

为实例绑定公钥

```bash
# 没有这个文件时，创建
touch /root/.ssh/authorized_keys

cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys

# 变更权限
chmod 600 /root/.ssh/authorized_keys
```

开通公钥认证功能

```bash
vim /etc/ssh/sshd_config
...
PubkeyAuthentication yes
...

# 重启服务
systemctl restart sshd
```

ssh终端连接

```bash
# 导出私钥文件到本地
sz id_rsa

# Workbench远程连接导入指定文件私钥内容
```

