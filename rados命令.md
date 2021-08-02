# rados命令

## pool相关

> 显示资源池列表
>
> ```shell
> rados lspools
> ```
>
> 创建资源池，默认pg_num和pgp_num为8。默认rule_id为0
>
> ```shell
> rados mkpool test
> ```
>
> 资源池数据拷贝
>
> ```shell
> rados cppool test test1
> ```
>
> 删除资源池
>
> ```shell
> rados rmpool test test --yes-i-really-really-mean-it
> ```
>
> 清除资源池数据
>
> ```shell
> rados purge test --yes-i-really-really-mean-it
> ```
>
> 查看资源池信息
>
> ```shell
> rados df -p test
> ```
>
> 列出资源池对象编号
>
> ```shell
> rados ls -p test
> rbd_data.166b6b8b4567.000000000000000040
> rbd_data.166b6b8b4567.000000000000000006
> ······
> ```

## object相关

> 获取对象内容
>
> ```shell
> rados -p test get rbd_data.166b6b8b4567.000000000000000040 test.txt
> ```
>
> 指定文件写入到资源池
>
> ```shell
> rados -p test put obj_name test.txt
> ```
>
> 向指定对象追加内容
>
> ```shell
> rados -p test append obj_name test.txt
> ```
>
> 删除指定对象，--force-full：强制删除，不管对象处于何种状态
>
> ```shell
> rados -p test rm object_name [--force-full]
> ```
>
> 拷贝对象
>
> ```shell
> rados -p test cp object_name cp_object_name
> ```

## 导出资源池数据

> 将资源池所有对象数据输出到指定文件中。文件可通过 ```hexdum -C pool_bak``` 命令格式化输出。
>
> ```shell
> rados -p test export pool_bak
> ```
>
> 将资源池文件导入到指定资源池
>
> ```shell
> rados -p test_rep_pool import pool_bak
> ```