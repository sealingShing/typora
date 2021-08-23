# librbd 写流程

入口函数：

```c++
Image::aio_write2
  |-->ImageRequestWQ<I>::aio_write
  |   \-->ImageRequest<I>::send()
  |      \-->AbstractImageWriteRequest<I>::send_request()
  |         |-->Striper::file_to_extents()  //将image分片，块->对象的映射
  |         \-->send_object_requests()      //调用send_object_requests 各个对象各自处理
  |            \-->AbstractObjectWriteRequest<I>::send()
  |               \-->AbstractObjectWriteRequest<I>::pre_write_object_map_update()
  |                  \-->AbstractObjectWriteRequest<I>::write_object()
  |                     \-->AbstractObjectWriteRequest<I>::handle_write_object()
  |                        \-->AbstractObjectWriteRequest<I>::post_write_object_map_update()
  |-->ImageRequestWQ<I>::queue
      \-->ImageRequestWQ<I>::process
```

类ImageWriteRequest继承自AbstractImageWriteRequest

librbd将请求发送给librados处理。





