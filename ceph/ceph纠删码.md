# ceph纠删码实践

```shell
#创建纠删码配置文件
ceph osd erasure-code-profile set Ecprofile crush-failure-domain=osd k=3 m=2
#查看纠删码配置文件
ceph osd erasure-code-profile ls
ceph osd erasure-code-profile get Ecprofile
ceph osd erasure-code-profile get default
#创建erasure类型的pool
ceph osd pool create Ecpool 16 16 erasure Ecprofile
#查看pool详细信息
ceph osd pool ls detail
#往pool中写数据
rados
```



ceph osd erasure-code-profile set Ecprofile crush-failure-domain=osd k=3 m=2

