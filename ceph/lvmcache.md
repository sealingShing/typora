# 基于lvmcache部署ceph步骤

## 硬件环境描述

一共三台服务器，每台服务器配置：

| 名称   | 配置                             |
| ------ | -------------------------------- |
| 处理器 | 鲲鹏920                          |
| 核数   | 96核                             |
| 内存   | 512GB                            |
| 硬盘   | 960G SSD * 2<br />1.1T   HDD * 6 |
| 网络   | 万兆                             |

## 软件

| 软件     | 版本                            |
| -------- | ------------------------------- |
| OS       | 4.19.90-24.1.v2101.ky10.aarch64 |
| ceph     | 13.2.5                          |
| ceph-cmd | d313b1f                         |

## 节点信息

| 主机类型    | 主机名称    | public网段 | cluster网段 |
| ----------- | ----------- | ---------- | ----------- |
| osd/mon/mgr | controller3 | 10.0.0.0/8 | 10.0.0.0/8  |
| osd/mon/mgr | compute1    | 10.0.0.0/8 | 10.0.0.0/8  |
| osd/mon/mgr | kvmtest1    | 10.0.0.0/8 | 10.0.0.0/8  |

## 创建lvmcache

### 磁盘分区



```shell
fdisk /dev/sda
```

删除lv

```shell
for i in `lvscan  | grep "vg/data_" | awk -F"'" '{print $2}'`;do yes y | lvremove $i;done
```

删除pv

```shell
for i in `pvscan | grep -Ev VG | awk '{print $2}'`;do pvremove $i;done
```



