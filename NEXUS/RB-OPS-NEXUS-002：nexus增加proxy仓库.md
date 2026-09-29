# NEXUS-OPS-002：nexus增加proxy仓库 Runbook

> **Metadata**
>
> - 环境: 赋能
>
> - 服务名称: `nexus`
> - 问题: `缺失依赖`
> - 负责人: @sre-oncall
> - 最后更新: 2026-07-17
> - 下次演练: xxx
> - 预计耗时: 30min
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 开发反馈：前端构建缺失依赖

## 前置检查清单

- [x] 已连接jumpserver
- [x] 登录到对应的服务器nexus
- [x] 已打开监控大盘：http://192.168.13.131:3000/
- [x] 已通知值班群：`前端构建缺失依赖，@oncall」`



## 快速诊断

### 1 排查进程

```bash
ps aux | grep nexus
```



### 2 查看磁盘容量

```bash
# 查看磁盘容量占用信息，对应的分区接近100%
df -h

# 查看详细数据占用目录
du -sh /*
```

### 3 查看Nexus中目标仓库

```bash
# 进入开发指定的依赖repository group查看，没有所需要的依赖
# 进入到group的仓库，查看是否有对应的依赖
```

### 4 增加proxy依赖仓库

1. 登录进入Nexus 
2. 进入设置Setting
3. 选择Repositories
4. Create Repositories
5.  选择仓库类型Recipe
6. 选择对应的类型，如npm(proxy)
7. 配置：Name; Proxy Remote storage 添加代理仓库（如：https://registry.npmjs.org）即可
8. 找到或者创建仓库类型Recipe如npm(group)，将代理仓库（如：https://registry.npmjs.org）添加到Members



### 5 下载依赖路径

```bash
npm install lodash
    ↓
请求发到 iot-nodejs-group（仅做路由判断）iot-nodejs-group（group）不存储任何数据，只做路由转发
    ↓
"lodash 不是内部包，转发到 iot-nodejs-taobao"
    ↓
iot-nodejs-taobao （proxy） 向淘宝镜像请求 lodash 或者 iot-nodejs（hosted）
    ↓													↓
下载成功后，缓存到 proxy 自己的 Blob Store 磁盘目录		有自己的 Blob Store 磁盘目录，存你上传的私有包
    ↓
返回给客户端
```

在磁盘上怎么看：

在 Nexus 服务器上，你可以通过以下方式确认存储位置：

1. **查看 Blob Store 配置**
   - 进入 **Server Administration → Repository → Blob Stores**
   - 找到 `iot-nodejs-taobao` 使用的 Blob Store（通常默认是 `default` 或自定义的）
   - 路径类似：`/nexus-data/blobs/default/` 或你配置的路径
2. **proxy 缓存的组件**
   - 进入 **Browse → iot-nodejs-taobao**
   - 可以看到所有已缓存的 npm 包及其版本
