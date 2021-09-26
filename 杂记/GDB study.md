## GDB study ##

| 调试指令                       | 作用                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| (gdb) break xxx \| (gdb) b xxx | 在源代码指定的某一行设置断点，其中xxx用于指定打断点的位置。xxx可以是代码行数也可以是函数名。 |
| (gdb) run \| (gdb) r           | 执行被调试程序，其会在第一个断点处暂停执行。                 |
| (gdb) continue \| (gdb) c      | 当程序在某一断点停止运行后，只用该指令可以继续执行，知道遇到下一个断点或者程序结束。 |
| (gdb) next \| (gdb) n          | 令程序一行一行代码的执行。                                   |
| (gdb) print xxx \| (gdb) p xxx | 打印指定变量的值或者设置函数参数，其中xxx指的某一变量名或者是函数名。 |
| (gdb) list \| (gdb) l          | 显示程序源码内容，包括行号。                                 |
| (gdb) quit \| (gdb) q          | 中止调试。                                                   |



```shell
#gdb调试子进程
[root@source1 ~]# gdb --args ceph-osd -i 0
(gdb) set follow-fork-mode child  #调试程序子进程
(gdb) br /data/ceph/src/osd/OpQueueItem.cc:24 #设置断点
(gdb) bt  #打印当前的函数调用栈
```

