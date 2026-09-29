# harbor证书过期更换Runbook

> **Metadata**
>
> - 环境: 生产
>
> - 服务名称: `kuboard`
> - 告警名称: `连接异常`
> - 负责人: @sre-oncall
> - 最后更新: 2026-09-15
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 用户反馈：Web页面无法正常显示

## 前置检查清单

- [x] 已连接进入ecs
- [x] 进入docker部署的服务
- [x] 已通知值班群：`「<service-name> 系统繁忙故障排查中，@oncall」`

## 操作步骤

```bash
1.进入harbor目录
cd /data/harbor

2.停止现在运行的harbor
docker-compose down

3.进入nginx的证书存放路径
ls
cd /data/harbor/download/Nginx/
ls

4.旧证书移除
mkdir 2026_03_25.bak
mv tian-power.cloud.* 2026_03_25.bak/
ls

7.将上传的文件移到/data/harbor/download/Nginx/目录下

8.mv /tmp/tian-power.cloud.* .
cd - 

#重载配置
9../prepare

10.起服务
docker-compose up -d
```

