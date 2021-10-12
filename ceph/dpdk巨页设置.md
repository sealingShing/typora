# dpdk巨页设置

```shell
 #关闭巨页
 echo 0 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
 echo 0 > /sys/kernel/mm/hugepages/hugepages-524288kB/nr_hugepages
 
 #开启巨页
 echo 16 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
 echo 16 > /sys/kernel/mm/hugepages/hugepages-524288kB/nr_hugepages
 
 #查看巨页设置情况
 cat /proc/meminfo  | grep -i huge
 
[root@localhost dpdk-19.11]# mount | grep huge
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel,pagesize=512M)
none on /mnt/huge type hugetlbfs (rw,relatime,seclabel,pagesize=512M)  ---挂了多个512M
[root@localhost dpdk-19.11]# mkdir -p /mnt/huge_2M
[root@localhost dpdk-19.11]# umount /mnt/huge
[root@localhost dpdk-19.11]# mount | grep huge
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel,pagesize=512M)
[root@localhost dpdk-19.11]# mount -t hugetlbfs none /mnt/huge_2M -o pagesize=2MB
[root@localhost dpdk-19.11]# mount | grep huge
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel,pagesize=512M)
none on /mnt/huge_2M type hugetlbfs (rw,relatime,seclabel,pagesize=2M)
[root@localhost dpdk-19.11]# 
```

