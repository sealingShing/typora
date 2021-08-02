# OSD write I/O



![image-20210726143156166](C:\Users\13729\AppData\Roaming\Typora\typora-user-images\image-20210726143156166.png)

通过ceph Messenger层的操作后，Dispatcher(OSD)对消息进行快速派发，将message存入工作队列中，OSD::ShardedOpWQ::_process()就是处理PG操作的核心函数。PrimaryLogPG::do_request()函数主要是做一些PG级别的检查，比如scan、scrub、backfill最终大部分普通操作都会进入PrimaryLogPG::do_op()函数。创建OpContext，最后调用PrimaryLogPG::execute_ctx()执行。

![PrimaryLogPG_execute_ctx](D:\typora\workspace\img\PrimaryLogPG_execute_ctx.png)

该函数是由do_op调用的， 主要工作是检查对象状态和上下文相关信息的获取，并调用函数prepare _transactions 把操作封装到事务中。如果是读取操作，则调用相关读取函数（同步、异步）。如果是写操作，则 调用calc_trim_to计算是否将旧的PG log日志进行trim操作、 issue_repop(repop, ctx)向各个副本发送同步操作请求、eval_repop(repop)检查发向各个副本的同步操作请求是否已经reply成功

## ceph osd 写流程

![write_3_replica](D:\typora\workspace\img\write_3_replica.png)

- Client  写数据，RADOS将数据发送给Primary OSD。
- Primary OSD识别Replica OSDs并且向他们发送数据，由他们写数据到本地磁盘。
- Replica OSDs 完成写并通知Primary OSD。
- Primary OSDs 通知client 写完成。

![write_op_message](D:\typora\workspace\img\write_op_message.jpg)

## issue_repop

![image-20210726154216610](C:\Users\13729\AppData\Roaming\Typora\typora-user-images\image-20210726154216610.png)

