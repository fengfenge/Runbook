# VM-OPS-001：VM虚拟机自定义磁盘资源Runbook

> **Metadata**
>
> - 环境: esxi
> - 服务名称: `vm`
> - 负责人: @idc
> - 最后更新: 2026-07-10
> - 下次演练: xxx
> - 预计耗时: 5-20 分钟

## 前置检查清单

- [x] 已连接ESXI
- [x] 登录到对应的服务器ESXI
- [x] 通知人：虚拟机运维人员



## 开通虚拟机

- 创建或注册虚拟机

- 选择名称和客户机
  - 名称和要部署的服务相关
  - 虚拟机兼容性：ESXI 8.0 U2虚拟机
  - 操作系统自选
- 选择存储
  - 根据磁盘选择
- 自定义设置
  - CPU/MEM 需要启动热添加
  - SCSI控制器0：VMware Paravirtual
  - usb控制器1：USB 2.0
  - 网络适配器：VM Network
  - CD/DVD驱动：数据存储ISO文件

### 1 引导固件BIOS

BIOS 是固化在电脑主板上的程序，主要用于开机系统自检和引导操作系统。目前新式的电脑基本上都是

UEFI启动

BIOS（Basic Input Output System 基本输入输出系统）主要完成系统硬件自检和引导操作系统，操作

系统开始启动之后，BIOS的任务就完成了。系统硬件自检：如果系统硬件有故障，主板上的扬声器就会

发出长短不同的“滴滴”音，可以简单的判断硬件故障，比如“1长1短”通常表示内存故障，“1长3短”通常

表示显卡故障

#### 1.1 自定义分区

自定义四个分区，如下：

| 名称         | device类型         | file system | 大小                 | 挂载点 |
| ------------ | ------------------ | ----------- | -------------------- | ------ |
| /dev/sda1    | Standard Partition | xfs         | 1G                   | /boot  |
| swap         | swap               | swap        | 2GiB                 | swap   |
| root         | Logical Volume     | xfs         | 25GiB                | /      |
| 数据分区/opt | Logical Volume     | xfs         | 剩余磁盘大小全部给它 | /opt   |

```bash
传统（普通）服务

磁盘大小为150G，自定义分区

boot 分1G

swap 分2G

/  分区分20G

/opt数据分区分 150-23=127G
```



```bash
k8s工作节点：

磁盘大小为150-200G

boot 1G

swap 2G

/  50G-80G

/opt数据分区 100 - 120G

hostname设置为 xxx.tian-power.com
```



### 2 引导固件EFI

EFI（Extensible Firmware Interface）可扩展固件接口，是 Intel 为PC 固件的体系结构、接口和服务提

出的建议标准。其主要目的是为了提供一组在 OS 加载之前（启动前）在所有平台上一致的、正确指定

的启动服务，被看做是BIOS的继任者，或者理解为新版BIOS。

#### 2.1 自定义分区

UEFI(Unified Extensible Firmware Interface)统一的可扩展固件接口， 是一种详细描述类型接口的标

准。UEFI相当于一个轻量化的操作系统，提供了硬件和操作系统之间的一个接口，提供了图形化的操作

界面。最关键的是引入了GPT分区表，支持2T以上的硬盘，硬盘分区不受限制



| 名称         | device类型         | file system          | 大小                 | 挂载点    |
| ------------ | ------------------ | -------------------- | -------------------- | --------- |
| /dev/sda1    | Standard Partition | EFI system partition | 1g                   | /boot/efi |
| /dev/sda2    | Standard Partition | xfs                  | 1G                   | /boot     |
| swap         | swap               | swap                 | 2GiB                 | swap      |
| root         | Logical Volume     | xfs                  | 25GiB                | /         |
| 数据分区/opt | Logical Volume     | xfs                  | 剩余磁盘大小全部给它 | /opt      |



```bash
普通服务

磁盘大小为150G，自定义分区

/boot 分1G

/boot/efi 分1G

swap 分2G

/  分区分20G

/opt数据分区分 150-23=127G
```

