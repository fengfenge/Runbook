# NEXUS-OPS-001：nexus数据迁移 Runbook

> **Metadata**
>
> - 环境: 赋能
>
> - 服务名称: `nexus`
> - 告警名称: `nexus服务器/根磁盘爆满`
> - 负责人: @sre-oncall
> - 最后更新: 2026-07-09
> - 下次演练: xxx
> - 预计耗时: 2 小时
> - 严重级别: P1（全量不可用）

---

## 触发条件

- [x] 开发反馈：服务不可用，数据无法上传

## 前置检查清单

- [x] 已连接jumpserver
- [x] 登录到对应的服务器nexus
- [x] 已打开监控大盘：http://192.168.13.131:3000/
- [x] 已通知值班群：`「<service-name> 系统繁忙故障排查中，@oncall」`



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



### 3 迁移数据

#### 3.1 确认数据目录位置

```bash
[root@nexus bin]# cat ./nexus.vmoptions | grep -E "karaf.data|nexus.data"
-Dkaraf.data=../sonatype-work/nexus3
```

#### 3.2 停服务

```bash
# 停止nexus服务
sudo systemctl stop nexus
# 或
sudo service nexus stop
# 或手动停止
/opt/nexus-3.78.2-04/bin/nexus stop
```

#### 3.3 查看数据目录属性

```bash
# 属主属组和读写权限
ll ../sonatype-work/nexus3
```

#### 3.3 创建新的数据目录并迁移数据

```bash
# 创建新目录
mkdir -p /opt/sonatype-work/

# 如果源目录的属主属组是nexus，则需要同步
chown -R nexus:nexus /opt/sonatype-work/
chmod 750 /opt/sonatype-work/

# 迁移数据（假设原数据在 /opt/sonatype-work/nexus3）
rsync -avP ./sonatype-work /opt/

# 验证同步完整性
du -sh ./sonatype-work/
du -sh /opt/sonatype-work/
```

#### 3.4 修改 Nexus 配置指向新目录

```bash
vim ./nexus-3.78.2-04/bin/nexus.vmoptions

# 原配置（类似这样）
-Dkaraf.data=../sonatype-work/nexus3

# 修改为
-Dkaraf.data=/opt/sonatype-work/nexus3
```

#### 3.5 验证及确认

```bash
# 验证元数据是否完整
[root@nexus nexus3]# rsync -avnc /opt/sonatype-work/nexus3/db/ /root/sonatype-work/nexus3/db/
sending incremental file list
nexus.mv.db

sent 94 bytes  received 19 bytes  226.00 bytes/sec
total size is 167,071,744  speedup is 1,478,511.01 (DRY RUN)

# 验证数据是否有差异。（迁移后启动了nexus，浏览了web，元数据可能有变化）
[root@nexus nexus3]# rsync -avnc --exclude="tmp/" --exclude="health-check/" /opt/sonatype-work/nexus3/blobs/ /root/sonatype-work/nexus3/blobs/
sending incremental file list
[root@nexus nexus3]# rsync -avnc --exclude="tmp/" --exclude="health-check/" /opt/sonatype-work/nexus3/blobs/ /root/sonatype-work/nexus3/blobs/
sending incremental file list
default/content/vol-10/chap-35/
default/content/vol-10/chap-35/05aef6b1-f159-472f-9394-4cc54294c10b.properties
default/content/vol-11/chap-35/
default/content/vol-15/chap-13/
default/content/vol-15/chap-13/b6e42b05-3863-4e31-baea-96de41006216.properties
default/content/vol-21/chap-36/
default/content/vol-21/chap-36/5d7b0431-8c16-43ab-88d1-152a95d03378.properties
default/content/vol-23/chap-44/
default/content/vol-23/chap-44/3bf0b316-ce44-4f0d-b4a3-93e32144b8ee.properties
default/content/vol-29/chap-21/
default/content/vol-29/chap-21/e0627d5e-2932-45ac-adc6-bca71ff6c71a.properties
default/content/vol-38/chap-22/
default/content/vol-38/chap-22/e53522c9-4cb1-4542-89ce-55ef9d11906d.properties
default/content/vol-40/chap-30/
default/content/vol-40/chap-30/01a76cd0-ef25-4c56-b96a-72eff3beb1ac.properties
default/reconciliation/
default/reconciliation/2026-07-10

sent 14,692,591 bytes  received 3,967 bytes  11,112.71 bytes/sec
total size is 38,665,491,618  speedup is 2,630.92 (DRY RUN)

# 权限和属主属组确认
ll /opt/sonatype-work

# 启动服务nexus
systemctl start nexus
或
nohup ./nexus run &

# 查看日志
tail -f /opt/sonatype-work/log/nexus.log

```

#### 3.6  验证正常后，清理旧数据（可选，建议保留一段时间）

```bash
# 确认一切正常后，可以压缩备份旧数据

tar czf nexus_data_backup_$(date +%Y%m%d).tar.gz sonatype-work/

```

