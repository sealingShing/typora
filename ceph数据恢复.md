# ceph数据恢复

## 前言

记录手动模拟ceph bluestore集群故障，osd数据为损坏，rbd-image恢复方法。
由于bluestore没有文件系统，不能像filestore那样直接在每个osd下的current目录下找到对应的对象数据文件，并且bluestore集群不可用，无法通过rados命令拿到pool中的对象数据。**osd处于离线状态。**

下面仅介绍数据恢复的流程，必要情况下可自己编写脚本进行操作。

## 测试集群准备

1、首先部署ceph v13.2.5容器版。创建一个pool，建立rbd池，创建一个image。测试环境是3台虚拟机，每台虚拟机有3个osd，集群默认3副本。

```sh
#创建pool rbd池
[ceph-cmd][d64f5f5][source]:~# ceph osd pool create rbd 128
pool 'rbd' created
#在rbd池内创建image1
[ceph-cmd][d64f5f5][source]:~# rbd create rbd/image1 --size 1G --image-feature layering
#查看rbd池内的image
[ceph-cmd][d64f5f5][source]:~# rbd ls -p rbd
image1
```

2、将image1块映射成块设备/dev/rbd0，将/dev/rbd0格式化成xfs格式，挂载到/mnt目录，向/mnt目录写入数据，再卸载/mnt目录，最后取消/dev/rbd0的映射。

```shell
#将image1映射成块设备
[ceph-cmd][d64f5f5][source]:~# rbd map rbd/image1
/dev/rbd0
#查看块设备映射信息
[ceph-cmd][d64f5f5][source]:~# rbd device ls
id pool image   snap  device
0  rbd  image1  -     /dev/rbd0
#格式化/dev/rbd0
[ceph-cmd][d64f5f5][source]:~# mkfs.xfs -f /dev/rbd0
#将/dev/rbd0挂载到/mnt目录
[ceph-cmd][d64f5f5][source]:~# mount /dev/rbd0 /mnt
#写入数据到/mnt目录
[ceph-cmd][d64f5f5][source]:~# cp /root/anaconda-ks.cfg /mnt/
#卸载/mnt目录
[ceph-cmd][d64f5f5][source]:~# umount /mnt
#取消image1与/dev/rbd0的映射关系
[ceph-cmd][d64f5f5][source]:~# rbd unmap rbd/image1
#查看块设备映射信息，为空。
[ceph-cmd][d64f5f5][source]:~# rbd device ls
```

3、记录集群所有信息

```shell
#获取pool数据
[ceph-cmd][d64f5f5][source]:~# ceph osd pool ls detail > /root/pool_info
#获取rbd images数据
[ceph-cmd][d64f5f5][source]:~# mkdir -p /root/rbd_info
[ceph-cmd][d64f5f5][source]:~# for pool in $(ceph osd pool ls);do for image in `rbd ls -p ${pool}`;do rbd info -p ${pool} ${image} > /root/rbd_info/${pool}_${image};done;done
```

4、停止所有osd相关容器服务，模拟bluestore集群损坏，但osd数据未损坏。 **通过ceph-objectstore-tool工具查看和获取对象时需要停止osd容器，该工具会挂载osd对应的磁盘。**

```shell
#停止所有osd容器服务，模拟bluestore集群损坏。
[ceph-cmd][d64f5f5][source]:~# daemon-stop host=* target=osd
```

## 数据收集

利用ceph-objectstore-tool工具收集osd数据

**注意：收集osd数据之前，先要关闭相应的osd服务**

```shell
#获取osd的data数据
[ceph-cmd][d64f5f5][source]:~# docker ps -a | grep ceph-osd | awk '{print $1}'
096577613b15
046c851096d7
[ceph-cmd][d64f5f5][source]:~# docker inspect 096577613b15 | python -c "import sys,json;mounts = json.load(sys.stdin)[0]['Mounts'];print [i for i in mounts if ['Destination'] == '/ceph/data'][0]['Source']"
/var/lib/docker/volumes/250...f87/_data
#获取osd的dev数据
[ceph-cmd][d64f5f5][source]:~# docker inspect 096577613b15 | python -c "import sys,json;mounts = json.load(sys.stdin)[0]['Mounts'];print [i for i in mounts if ['Destination'] == '/ceph/dev'][0]['Source']"
/var/lib/docker/volumes/ef7...8a9/_data
#拷贝osd的data数据和dev数据
[ceph-cmd][d64f5f5][source]:~# cp -r /var/lib/docker/volumes/250...f87/_data /root/096577613b15_data
[ceph-cmd][d64f5f5][source]:~# cp -r /var/lib/docker/volumes/ef7...8a9/_data /ceph/dev
[ceph-cmd][d64f5f5][source]:~# for main in $(ls /ceph/dev/_data);do mv /ceph/dev/_data/${main} /ceph/dev

#获取osd内object数据列表
[ceph-cmd][d64f5f5][source]:~# ceph-objectstore-tool --no-mon-config --data-path /root/096577613b15_data --op list > /root/096577613b15_object_list
[ceph-cmd][d64f5f5][source]:~# cat /root/096577613b15_object_list
["2.3a",{"oid":"rbd_data.166b6b8b4567.0000000000000001","key":"","snapid":-2,"hash":235010478,"max":0,"pool":1,"namespace":"","max":0}]
......
["1.e",{"oid":"rbd_data.166b6b8b4567.0000000000000010","key":"","snapid":-2,"hash":281238148,"max":0,"pool":1,"namespace":"","max":0}]

#导出osd内object数据
[ceph-cmd][d64f5f5][source]:~# ceph-objectstore-tool --no-mon-config --data-path /root/096577613b15_data ["2.3a",{"oid":"rbd_data.166b6b8b4567.0000000000000001","key":"","snapid":-2,"hash":235010478,"max":0,"pool":1,"namespace":"","max":0}] get-bytes rbd_data.166b6b8b4567.0000000000000001
......
[ceph-cmd][d64f5f5][source]:~# ceph-objectstore-tool --no-mon-config --data-path /root/096577613b15_data ["1.e",{"oid":"rbd_data.166b6b8b4567.0000000000000010","key":"","snapid":-2,"hash":281238148,"max":0,"pool":1,"namespace":"","max":0}] get-bytes rbd_data.166b6b8b4567.0000000000000010
```

根据上面的方法，可以将每个osd的object数据导出来 。**多副本集群需要保证数据一致性，所有数据导出后，再进行去重。整合后获得集群所有object数据。**

## 数据恢复

根据上面数据获取的数据，创建ceph集群。新建同名pool以及同名images。

**注意：pool的pg num可以不用完全一样。如果有snapshot，需要手动创建快照，与原集群的关系保持一致。clone关系一样。**

待新集群创建完成后。**在原数据中把对应的位置全部替换（即把旧image对象中带有rbd_id的地方替换为新的rbd_id，然后再导入新pool中，比如rdb_data.xxxxxx.000000002中间的rbd_id需要替换为新的，rbd_object_map也需要替换，rbd_header的rbd_id都替换），没带rbd_id的对象数据可不必导入到新集群中。**

```shell
#数据导入到相应的pool中
[ceph-cmd][d64f5f5][source]:~# rados -p pool_name put rdb_data.16636b9b4568.0000000000000002 /recover_data/rdb_data.16636b9b4568.0000000000000002
```

最后验证数据。可将image映射成块设备，挂载后验证内部文件。

----------------------------------------------------------------------------------------------------------------------------------------------------

## 更新 ##

集群出现

1. ```shell
   [ceph-cmd][d64f5f5][host156]:/# ceph -s  
     cluster:
       id:     3d3dc1d1-acf6-40da-8fc3-6de7e733bd7c
       health: HEALTH_ERR
               6 scrub errors
               Possible data damage: 2 pgs inconsistent
    
     services:
       mon:      3 daemons, quorum host156,host157,host158
       mgr:      host157(active), standbys: host156, host158
       osd:      6 osds: 6 up, 6 in
       exporter: 1 daemon active
    
     data:
       pools:   4 pools, 272 pgs
       objects: 35  objects, 47 MiB
       usage:   6.3 GiB used, 114 GiB / 120 GiB avail
       pgs:     270 active+clean
                2   active+clean+inconsistent
   [ceph-cmd][d64f5f5][host156]:/# ceph pg repair 1.c
   instructing pg 1.c on osd.3 to repair
   [ceph-cmd][d64f5f5][host156]:/# ceph -w           
     cluster:
       id:     3d3dc1d1-acf6-40da-8fc3-6de7e733bd7c
       health: HEALTH_ERR
               6 scrub errors
               Possible data damage: 2 pgs inconsistent
    
     services:
       mon:      3 daemons, quorum host156,host157,host158
       mgr:      host157(active), standbys: host156, host158
       osd:      6 osds: 6 up, 6 in
       exporter: 1 daemon active
    
     data:
       pools:   4 pools, 272 pgs
       objects: 35  objects, 47 MiB
       usage:   6.3 GiB used, 114 GiB / 120 GiB avail
       pgs:     270 active+clean
                2   active+clean+inconsistent
    
   
   2021-04-27 14:32:44.932639 osd.3 [ERR] 1.c shard 2 soid 1:3072a5b7:::rbd_data.13a46b8b4567.0000000000000101:4 : candidate had a missing info key
   2021-04-27 14:32:44.932641 osd.3 [ERR] 1.c soid 1:3072a5b7:::rbd_data.13a46b8b4567.0000000000000101:4 : failed to pick suitable object info
   2021-04-27 14:32:44.932676 osd.3 [ERR] 1.c repair 3 errors, 0 fixed
   ```

