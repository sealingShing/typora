# bcache安装部署

## 安装bcache

```shell
#安装依赖libblkid-devel
yum install -y libblkid-devel

#下载源码
[root@source1 ~]# git clone https://github.com/g2p/bcache-tools.git
#编译安装
[root@source1 ~]# cd bcache-tools
[root@source1 bcache-tools]# make && make install
cc -O2 -Wall -g `pkg-config --cflags uuid blkid`   -c -o bcache.o bcache.c
bcache.c:125:9: 警告：‘crc_table’是静态的，但却在非静态的内联函数‘crc64’中被使用
   crc = crc_table[i] ^ (crc << 8);
         ^~~~~~~~~
cc -O2 -Wall -g `pkg-config --cflags uuid blkid`    make-bcache.c bcache.o  `pkg-config --libs uuid blkid` -o make-bcache
/usr/bin/ld: /tmp/ccXYWfRd.o: in function `write_sb':
/root/bcache-tools/make-bcache.c:277: undefined reference to `crc64'
collect2: 错误：ld 返回 1
make: *** [<内置>：make-bcache] 错误 1


#解决上述问题
--- a/bcache.c
+++ b/bcache.c
@@ -115,7 +115,7 @@ static const uint64_t crc_table[256] = {
    0x9AFCE626CE85B507ULL
 };
 
-inline uint64_t crc64(const void *_data, size_t len)
+uint64_t crc64(const void *_data, size_t len)
 {
    uint64_t crc = 0xFFFFFFFFFFFFFFFFULL;
    const unsigned char *data = _data;
    
https://github.com/g2p/bcache-tools/issues/30
```

## 配置使用

### 创建后端低速设备

```shell
[root@source1 ~]# make-bcache -B /dev/sdf
Device /dev/sdf already has a non-bcache superblock, remove it using wipefs and wipefs -a
[root@source1 ~]# wipefs -a /dev/sdf
/dev/sdf：4 个字节已擦除，位置偏移为 0x00000000 (xfs)：58 46 53 42
[root@source1 ~]# make-bcache -B /dev/sdf
UUID:			cb31c516-9a68-4afc-a4e9-3e6d82696684
Set UUID:		b17c14f8-0f7f-4eb8-8444-7017d473f859
version:		1
block_size:		1
data_offset:	16
```

### 创建前端缓存磁盘（SSD）

```shell
[root@source1 ~]# make-bcache -C /dev/sdb
UUID:			a1d2fd02-e7f8-460f-af51-3e69c2c4a03f
Set UUID:		7a47558e-4558-4793-b302-214654431686
version:		0
nbuckets:		8089
block_size:		1
bucket_size:	1024
nr_in_set:		1
nr_this_dev:	0
first_bucket:	1
```

### 建立映射关系

```shell
#获取cset.uuid
[root@source1 ~]# bcache-super-show /dev/sdd
sb.magic		ok
sb.first_sector		8 [match]
sb.csum			B308BCB3DC9830EF [match]
sb.version		0 [cache device]

dev.label		(empty)
dev.uuid		1a942457-f873-4459-8826-7bb0732dfd99
dev.sectors_per_block	1
dev.sectors_per_bucket	1024
dev.cache.first_sector	1024
dev.cache.cache_sectors	8387584
dev.cache.total_sectors	8388608
dev.cache.ordered	no
dev.cache.discard	no
dev.cache.pos		0
dev.cache.replacement	0 [lru]

cset.uuid		beb4ae04-74a0-456d-915b-619c4efdd88a
#建立联系
[root@source1 ~]# echo "beb4ae04-74a0-456d-915b-619c4efdd88a" > /sys/block/bcache2/bcache/attach

#建立连接后
[root@source1 ~]# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda             8:0    0   80G  0 disk 
├─sda1          8:1    0  200M  0 part /boot/efi
├─sda2          8:2    0    1G  0 part /boot
└─sda3          8:3    0 78.8G  0 part 
  ├─klas-root 252:0    0 74.8G  0 lvm  /
  └─klas-swap 252:1    0    4G  0 lvm  [SWAP]
sdb             8:16   0    4G  0 disk 
└─bcache0     251:0    0   20G  0 disk 
sdc             8:32   0    4G  0 disk 
└─bcache1     251:128  0   20G  0 disk 
sdd             8:48   0    4G  0 disk 
└─bcache2     251:256  0   20G  0 disk 
sde             8:64   0   60G  0 disk 
sdf             8:80   0   20G  0 disk 
└─bcache0     251:0    0   20G  0 disk 
sdg             8:96   0   20G  0 disk 
└─bcache1     251:128  0   20G  0 disk 
sdh             8:112  0    4G  0 disk 
sdi             8:128  0   20G  0 disk 
└─bcache2     251:256  0   20G  0 disk 

```

### 快捷建立连接方式

```shell
[root@source1 ~]# make-bcache -C /dev/sde -B /dev/sdb 
```

### 修改缓存策略

```shell
[root@source1 wfs_test]# echo writeback > /sys/block/bcache2/bcache/cache_mode
[root@source1 wfs_test]# cat /sys/block/bcache1/bcache/cache_mode 
writethrough [writeback] writearound none
```

## 删除bcache配置

### 删除前端缓存盘

```shell
[root@source1 ~]# echo 1 > /sys/fs/bcache/1e6430e6-2670-45f9-b0e4-f2be52b4010a/unregister
```

### 删除后端盘

```shell
[root@source1 ~]# echo 1 > /sys/block/bcache0/bcache/stop
[root@source1 ~]# echo 1 > /sys/block/bcache1/bcache/stop
[root@source1 ~]# echo 1 > /sys/block/bcache2/bcache/stop
```

