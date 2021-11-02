ceph v13.2.10源码编译安装

``注意：虚拟机资源多给一些。16C16G``

## 编译环境

cpu：16C

内存：16G

```shell
[root@source ~]# uname -a
Linux source 4.19.90-9.ky10.aarch64 #1 SMP Sun Apr 26 11:05:59 CST 2020 aarch64 aarch64 aarch64 GNU/Linux
```



## 依赖安装

```shell

yum install git gcc-c++ cmake  gperftools-devel  jemalloc-devel python-sphinx systemd-devel fuse-devel xfsprogs-devel libaio-devel snappy-devel lz4-devel keyutils-libs-devel curl-devel nss-devel openssl-devel expat-devel  lttng-ust-devel libbabeltrace-devel python2-prettytable python3-prettytable python2-Cython python3-Cython gperf CUnit-devel python3-devel

```

## 源码下载

```shell
git clone --recursive https://gitee.com/mirrors/ceph.git
git checkout v13.2.10
git submodule update --init --recursive
```

当子模块总是下载失败时：

在git clone的地址，例如https://github.com/pytorch/pytorch，改为https://github.com.cnpmjs.org/pytorch/pytorch，也即加上后缀.cnpmjs.org，然后就可以愉快的下载了(亲测有效)。

对于子模块，可以先不要在git clone的时候加上--recursive，等主体部分下载完之后，该文件夹中有个隐藏文件称为：.gitmodules，把子项目中的url地址同样加上.cnpmjs.org后缀，然后利用git submodule sync更新子项目对应的url，最后再git submodule update --init --recursive，即可正常网速clone完所有子项目。

## 编译

```shell
cd /data/ceph/
mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/ceph -DWITH_RDMA=OFF -DWITH_OPENLDAP=OFF -DWITH_LEVELDB=OFF -DMGR_PYTHON_VERSION=2 -DWITH_PYTHON2=ON -DWITH_RADOSGW=OFF -DBOOST_J=4 ..

```

```shell
#编译开启DPDK和SPDK
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/ceph -DWITH_RDMA=OFF -DWITH_OPENLDAP=OFF -DWITH_LEVELDB=OFF -DMGR_PYTHON_VERSION=2 -DWITH_PYTHON2=ON -DWITH_RADOSGW=OFF -DBOOST_J=4 -DWITH_DPDK=ON -DWITH_SPDK=ON ..

#编译遇到的问题1
  Could NOT find CUnit (missing: CUNIT_LIBRARY CUNIT_INCLUDE_DIR)
  #解决
    yum install CUnit-devel

#编译遇到的问题2 
  Could NOT find dpdk (missing: DPDK_INCLUDE_DIR check_LIBRARIES)
  #解决
    编译安装dpdk
#编译遇到的问题3 
  /data/ceph/src/msg/async/dpdk/TCP.h:36:10: 致命错误：cryptopp/md5.h：没有那个文件或目录
  #解决 
    cd ceph/src
    git clone https://github.com/weidai11/cryptopp.git
    
#编译遇到的问题4
[root@source build]# cmake -DCMAKE_INSTALL_PREFIX=/usr/local/ceph -DWITH_RDMA=OFF -DWITH_OPENLDAP=OFF -DWITH_LEVELDB=OFF -DMGR_PYTHON_VERSION=2 -DWITH_PYTHON2=ON -DWITH_RADOSGW=OFF -DBOOST_J=4 ..
-- NSS_LIBRARIES: /usr/lib64/libssl3.so;/usr/lib64/libsmime3.so;/usr/lib64/libnss3.so;/usr/lib64/libnssutil3.so
-- NSS_INCLUDE_DIRS: /usr/include/nss3
-- Found PythonLibs: /usr/lib64/libpython2.7.so (found suitable version "2.7.16", minimum required is "2") 
-- BUILDING Boost Libraries at j 4
-- boost will be downloaded...
-- We are using libstdc++.
--  we do not have a modern/working yasm
-- Found PythonLibs: /usr/lib64/libpython2.7.so (found suitable exact version "2.7.16") 
-- Found Python3Libs: /usr/lib64/libpython3.7m.so (found suitable exact version "3.7.4") 
--  Using EventEpoll for events.
-- Found cython3
-- Found cython
-- Found PythonInterp: /usr/bin/python2 (found version "2.7.16") 
-- exclude following files under src: *.js;*.css;civetweb;erasure-code/jerasure/jerasure;erasure-code/jerasure/gf-complete;rocksdb;googletest;spdk;xxHash;isa-l;lua;zstd;crypto/isa-l/isa-l_crypto;blkin;rapidjson
CMake Error: The following variables are used in this project, but they are set to NOTFOUND.
Please set them or make sure they are set and tested correctly in the CMake files:
DPDK_rte_pmd_vmxnet3_uio_LIBRARY
    linked by target "common_async_dpdk" in directory /data/ceph/src

#问题解决
  vim /data/ceph/build/src/CMakeFiles/ceph-common.dir/link.txt
  加上-lcryptopp
  + -lblkid -lssl3 -lsmime3 -lnss3 -lnssutil3 -lplds4 -lplc4 -lnspr4 -lssl -lcrypto -lcryptopp -lpthread -ldl -lz
  
#问题5
/data/ceph/src/msg/async/dpdk/dpdk_rte.h:38:2: 警告：#warning "CONFIG_RTE_MBUF_REFCNT_ATOMIC should be disabled in DPDK's " "config/common_linuxapp" [-Wcpp]
 #warning "CONFIG_RTE_MBUF_REFCNT_ATOMIC should be disabled in DPDK's " \

```



编译补充: 手动下载boost_1_67_0.tar.bz2到
build/boost/src

https://boostorg.jfrog.io/artifactory/main/release/1.67.0/source/boost_1_67_0.tar.bz2

编译补充: 还需要下载
liboath-2.6.2-1.el7.aarch64.rpm  和liboath-devel-2.6.2-1.el7.aarch64.rpm两个包。 解决liboath的错误

https://download-ib01.fedoraproject.org/pub/epel/7/aarch64/Packages/l/liboath-devel-2.6.2-1.el7.aarch64.rpm

https://download-ib01.fedoraproject.org/pub/epel/7/aarch64/Packages/l/liboath-2.6.2-1.el7.aarch64.rpm

源码打入patch

```shell
cd /data/ceph
patch -p1 < /root/checkpoint-on-librbd-queue-num.patch
cd /data/ceph/build
make -j10
make install
```

## 裸机部署

### 环境变量设置

在/usr/lib/python2.7/site-packages写入python.pth文件
/usr/local/ceph/lib/python2.7/site-packages
/usr/local/ceph/lib64/python2.7/site-packages

在/etc/ld.so.conf中添加
/usr/local/ceph/lib64
执行ldconfig

在/etc/profile末尾添加。
PATH=$PATH:/usr/local/ceph/bin
执行source /etc/profile



