# ceph cache tier



## 配置缓存池

### 创建缓存池

```shell
#查看osd的class类型
[root@host124 testceph]# ceph osd crush class ls
[
    "hdd",
    "ssd"
]
#查看当前osd布局
[root@host124 testceph]# ceph osd tree
ID CLASS WEIGHT   TYPE NAME        STATUS REWEIGHT PRI-AFF 
-1       13.13425 root default                             
-3        4.37808     host host123                         
 0   hdd  1.09160         osd.0        up  1.00000 1.00000 
 1   hdd  1.09160         osd.1        up  1.00000 1.00000 
 2   hdd  1.09160         osd.2        up  1.00000 1.00000 
 3   hdd  1.09160         osd.3        up  1.00000 1.00000 
12   ssd  0.01169         osd.12       up  1.00000 1.00000 
-5        4.37808     host host124                         
 4   hdd  1.09160         osd.4        up  1.00000 1.00000 
 5   hdd  1.09160         osd.5        up  1.00000 1.00000 
 6   hdd  1.09160         osd.6        up  1.00000 1.00000 
 7   hdd  1.09160         osd.7        up  1.00000 1.00000 
13   ssd  0.01169         osd.13       up  1.00000 1.00000 
-7        4.37808     host host183                         
 8   hdd  1.09160         osd.8        up  1.00000 1.00000 
 9   hdd  1.09160         osd.9        up  1.00000 1.00000 
10   hdd  1.09160         osd.10       up  1.00000 1.00000 
11   hdd  1.09160         osd.11       up  1.00000 1.00000 
14   ssd  0.01169         osd.14       up  1.00000 1.00000 

#创建基于ssd的class rule
[root@host124 testceph]# ceph osd crush rule create-replicated ssd_rule default host ssd
[root@host124 testceph]# ceph osd crush rule ls 
replicated_rule
ssd_rule


#查看详细的crushmap规则
[root@host124 testceph]#  ceph osd getcrushmap -o crushmap
148
[root@host124 testceph]#  crushtool -d crushmap -o crushmap.txt
[root@host124 testceph]# cat crushmap.txt 
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd
device 6 osd.6 class hdd
device 7 osd.7 class hdd
device 8 osd.8 class hdd
device 9 osd.9 class hdd
device 10 osd.10 class hdd
device 11 osd.11 class hdd
device 12 osd.12 class ssd
device 13 osd.13 class ssd
device 14 osd.14 class ssd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host host123 {
	id -3		# do not change unnecessarily
	id -2 class hdd		# do not change unnecessarily
	id -9 class ssd		# do not change unnecessarily
	# weight 4.378
	alg straw2
	hash 0	# rjenkins1
	item osd.0 weight 1.092
	item osd.1 weight 1.092
	item osd.2 weight 1.092
	item osd.3 weight 1.092
	item osd.12 weight 0.012
}
host host124 {
	id -5		# do not change unnecessarily
	id -4 class hdd		# do not change unnecessarily
	id -10 class ssd		# do not change unnecessarily
	# weight 4.378
	alg straw2
	hash 0	# rjenkins1
	item osd.4 weight 1.092
	item osd.5 weight 1.092
	item osd.6 weight 1.092
	item osd.7 weight 1.092
	item osd.13 weight 0.012
}
host host183 {
	id -7		# do not change unnecessarily
	id -6 class hdd		# do not change unnecessarily
	id -11 class ssd		# do not change unnecessarily
	# weight 4.378
	alg straw2
	hash 0	# rjenkins1
	item osd.8 weight 1.092
	item osd.9 weight 1.092
	item osd.10 weight 1.092
	item osd.11 weight 1.092
	item osd.14 weight 0.012
}
root default {
	id -1		# do not change unnecessarily
	id -8 class hdd		# do not change unnecessarily
	id -12 class ssd		# do not change unnecessarily
	# weight 13.134
	alg straw2
	hash 0	# rjenkins1
	item host123 weight 4.378
	item host124 weight 4.378
	item host183 weight 4.378
}

# rules
rule replicated_rule {
	id 0
	type replicated
	min_size 1
	max_size 10
	step take default
	step chooseleaf firstn 0 type host
	step emit
}
rule ssd_rule {
	id 1
	type replicated
	min_size 1
	max_size 10
	step take default class ssd
	step chooseleaf firstn 0 type host
	step emit
}

# end crush map


#创建基于ssd_rule规则的存储池
[root@host124 testceph]# ceph osd pool create cache 128 128 ssd_rule
pool 'cache' created
```

### 设置缓存池

```shell
#将缓存池绑定到存储池前端
[root@host124 testceph]# ceph osd tier add volumes cache
pool 'cache' is now (or already was) a tier of 'volumes'

#设置缓存池为writeback模式
[root@host124 testceph]# ceph osd tier cache-mode cache writeback
set cache-mode for pool 'cache' to writeback

#将所有客户端请求从标准池引导至缓存池
[root@host124 testceph]# ceph osd tier set-overlay volumes cache
overlay for 'volumes' is now (or already was) 'cache'

#查看存储池详情
[root@host124 testceph]# ceph osd dump | egrep 'volumes|cache'
pool 5 'volumes' replicated size 1 min_size 1 crush_rule 0 object_hash rjenkins pg_num 512 pgp_num 512 last_change 5256 lfor 5256/5256 flags hashpspool,selfmanaged_snaps tiers 6 read_tier 6 write_tier 6 stripe_width 0 application rbd
pool 6 'cache' replicated size 1 min_size 1 crush_rule 1 object_hash rjenkins pg_num 128 pgp_num 128 last_change 5256 lfor 5256/5256 flags hashpspool,incomplete_clones,selfmanaged_snaps tier_of 5 cache_mode writeback stripe_width 0

```

### 缓存层相关参数说明

```shell
# 对于生产环境的部署，目前只能使用bloom filters数据结构（看官方文档的意思，好像目前只支持这一种filter）
[root@host124 testceph]# ceph osd pool set cache hit_set_type bloom
set pool 6 hit_set_type to bloom

# 设置当缓存池中的数据达到多少个字节或者多少个对象时，缓存分层代理就开始从缓存池刷新对象至后端存储池并驱逐
# 当缓存池中的数据量达到20G时开始刷盘并驱逐
[root@host124 testceph]# ceph osd pool set cache target_max_bytes 21474836480
set pool 6 target_max_bytes to 21474836480
# 当缓存池中的对象个数达到1万时开始刷盘并驱逐
[root@host124 testceph]# ceph osd pool set cache target_max_objects 10000
set pool 6 target_max_objects to 10000

#定义缓存层将对象刷至存储层或者驱逐的时
#驱逐对象的最小时间设置为60s
[root@host124 testceph]# ceph osd pool set cache cache_min_flush_age 60
set pool 6 cache_min_flush_age to 60
[root@host124 testceph]# ceph osd pool set cache cache_min_evict_age 60
set pool 6 cache_min_evict_age to 60

#定义当缓存池中的脏对象（被修改过的对象）占比达到多少时，缓存分层代理开始将object从缓存层刷至存储层
# 当脏对象占比达到10%时开始刷盘
[root@host124 testceph]# ceph osd pool set cache cache_target_dirty_ratio 0.1
set pool 6 cache_target_dirty_ratio to 0.1
# 当脏对象占比达到60%时开始高速刷盘
[root@host124 testceph]# ceph osd pool set cache cache_target_dirty_high_ratio 0.6
set pool 6 cache_target_dirty_high_ratio to 0.6
#当缓存池的使用量达到其总量的一定百分比时，缓存分层代理将驱逐对象以维护可用容量（达到该限制时，就认为缓存池满了），此时会将未修改的（干净的）对象刷盘
[root@host124 testceph]# ceph osd pool set cache cache_target_full_ratio 0.8
set pool 6 cache_target_full_ratio to 0.8

#启用hit set count,即缓存池中存储的hit set（命中集）的数量，数量越大，OSD占用的内存量就越大
ceph osd pool set cache hit_set_count 1
```

### 删除缓存池

```shell
#将缓存模式更改为转发，以便新的和修改的对象刷新至后端存储池：
[root@source1 ~]# ceph osd tier cache-mode cache forward

#查看缓存池以确保所有的对象都被刷新（这可能需要点时间）：
[root@source1 ~]# rados -p cache ls 
#如果缓存池中仍然有对象，也可以手动刷新：
[root@source1 ~]# rados -p cache cache-flush-evict-all


#删除覆盖层，以使客户端不再将流量引导至缓存
[root@source1 ~]# ceph osd tier remove-overlay cache
there is now (or already was) no overlay for 'cache'
[root@source1 ~]# ceph osd tier remove-overlay test
there is now (or already was) no overlay for 'test'

#解除绑定
[root@source1 ~]# ceph osd tier remove test cache 
pool 'cache' is now (or already was) not a tier of 'test'

#删除cache池
[root@source1 ~]# ceph osd pool rm cache cache  --yes-i-really-really-mean-it
pool 'cache' removed

```



