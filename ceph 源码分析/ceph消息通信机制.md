# ceph消息通信

ceph有3种消息通信框架：simple、xio、async。当前ceph mimic版本默认的消息通信框架是async

## 消息模块生命周期

 ![messenger生命周期.jfif](..\img\messenger生命周期.jfif) 

以osd为例描述消息模块的生命周期，在ceph-osd的守护进程main()函数中注册并创建一个Messenger，然后对注册的Messenger进行绑定，绑定后开启消息模块进行工作，消息模块启动后即osd初始化，在初始化的过程中让Messenger处于ready状态，即工作状态。当消息模块工作状态结束后处于wait状态，如果需要的话则删除注册的Messenger。

OSD注册的Messenger实例列表：

| 编号 | Messenger实例名称   | 作用                                 |
| ---- | ------------------- | ------------------------------------ |
| 1    | *ms_public          | 用来处理OSD和Client之间的消息        |
| 2    | *ms_cluster         | 用来处理OSD和集群之间的消息          |
| 3    | *ms_hb_back_client  | 用来处理OSD接收其它OSD保持心跳的消息 |
| 4    | *ms_hb_front_client | 用来处理OSD发送其它OSD保持心跳的消息 |
| 5    | *ms_hb_back_server  | 用来处理OSD接收心跳消息              |
| 6    | *ms_hb_front_server | 用来处理OSD发送心跳消息              |
| 7    | *ms_objecter        | 用来处理OSD和Objecter之间的消息      |



## 消息模块初始化

 ![messenger_init.jfif](..\img\messenger_init.jpg) 

首先在main()函数中执行启动当前节点需要的一些配置和初始化，其中包含Messenger Dispatcher的创建和初始化，mon/mgr/osd/mds/client等都是Dispatcher的子类。Messenger::create()用来创建一个消息处理者Messenger，实际上创建了AsyncMessenger实例。在main()函数中调用AsyncMessenger::bind()为每一个AsyncMessenger绑定一个用于网络传输的IP地址。在OSD::init()函数中调用add_dispatcher_head()，执行以下操作：

- 将osd创建的所有Dispatcher添加到Messenger的dispatcher队列中
- 调用AsyncMessenger::ready()启动AsyncMessenger

消息模块的初始化主要启动两个模块，一个是EventCenter(事件中心)，并一个是AsyncMessenger，调用AsyncMessenger::ready()获取一个工作线程，然后执行Processor::start()进行具体的初始化工作。在EpollDriver::create_file_events()中创建文件，调用EpollDriver::add_events()，执行epoll_ctl，启动向epoll注册事件，事件中心的EventCenter::process_events()在等待事件的产生。同事Processor::accept()也被执行来准备接受连接。​

## 事件中心的启动

![image-20210722200233178](C:\Users\13729\AppData\Roaming\Typora\typora-user-images\image-20210722200233178.png)

在AsyncMessenger网络模块中，采用事件驱动模型，在事件驱动模型中有一个事件处理中心用来处理注册的事件。下面介绍事件中心的初始化。

首先，在OSD守护进程中启动Messenger，创建一个single单例single::ready()实际上调用NetworkStack::create()，创建过程中会创建Workers，工作线程数量根据配置参数ms_async_op_threads : 3(默认值)创建。事件中心的工作都交给worker线程来执行。NetworkStack::start()启动协议栈，其主要工作就是针对每一个Workers[i]启动一个线程，这个线程的核心任务就是循环执行"Worker.EventCenter.process_events()"，相关的资源都被封装在Worker及其子类中。调用EpoolDriver的event_wait()函数执行，即epoll的主循环，返回需要处理事件的数量。用一个for循环处理epoll_wait()返回的事件，调用File *_get_file_event()函数创建一个文件事件，根据文件事件的mask判定是读还是写，然后执行do_request()进行具体的操作。看下外部事件容器中是否有事件需要处理，如果有，使用while循环处理，具体处理过程调用do_request()函数。

## 消息的接受

### 启动消息接受

 在消息初始化的模块中创建一个线程，专门用来处理事件。

![启动消息接受.jfif](..\img\启动消息接受.jpg)

在消息接收之前，事件中心已经启动， ，这时候还没有事件放到事件中心去处理。事件产生通过EventcEnter::create_file_event()进行的，所以消息的接收是以事件的形式操作的，然后调用相关的模块来接收消息。

在Processor::start()调用EventcEnter::create_file_event()会传递两个参数，一个是mask值EVENT_READABLE，告诉事件中心处理消息的读，另一个是回调指针listen_handler，实际上是实质是C_processor_accept实例。EventcEnter::create_file_event()接收的事件操作是C_processor_accept实例。判定mask，如果是EVENT_READABLE，调用相应的回调函数来处理事件，此时的回调类是C_processor_accept，执行的回调操作是接收连接，具体来说就是Processor::accept()，一方面调用标准socket函数accept接收连接，另一方面调用AsyncConnectionRef AsyncMessenger::add_accept()处理连接。AsyncMessenger::add_accept()处理请求时先创建了一个连接AsyncConnection，然后调用AsyncConnection的accept()来接收消息，先将state的值置为STATE_ACCEPTING，然后创建一个接收消息的事件让事件中心处理。事件的maks是EVENT_READABLE，回调操作是read_handler，如果这个时候创建事件这个操作被锁住了，则将消息读操作放入外部事件容器中，等到事件中心处理事件的函数解锁以后会去处理外部事件容器，也会继续处理消息的读操作。

### 消息接收工作流程

![image-20210723170243877](C:\Users\13729\AppData\Roaming\Typora\typora-user-images\image-20210723170243877.png)

 消息的处理有两个途径，一个是根据事件的mask判定事件是否为EVENT_READABLE，如果yes将read_cb回调指针指向传入的回调操作，即read_handler。另一个处理途径是当前create_file_event()正在执行别的操作被锁住了，则通过dispatch_event_external()函数将回调操作read_handler放入external_events中，事件处理中心有一个循环在轮询external_events，一旦发现有回调操作放入，则调用相应的回调函数来处理。这两个途径最后都是通过执行read_cb的回调函数来完成消息的读操作。

read_cb的回调操作是调用AsyncConnection::process()处理。在process()中有一个switch操作，根据之前accept()接收的state的值找到相应的执行体，根据state的值进入AsyncConnection::_process_connection()处理。在_process_connection()中新建了一个bufferlist，把CEPH_BANNER添加到bl中，CEPH_BANNER是一个字符串，标识这个消息是ceph的数据。然后通过get_myaddr()获取一个Messenger实例的地址encode到bl中。接着将socket_addr也encode到bl中，调用try_send()执行消息的发送准备工作，try_send()函数执行完成后返回剩余没发送的消息字节的长度，如果返回的值为0，说明消息发送完毕，将stated的值置为STATE_ACCEPTING_WAIT_BANNER_ADDR，如果返回的值大于0，说明消息没有发送完成，将state的值置为STATE_WAIT_SEND。

启动消息的接收以后，消息的接收状态是STATE_ACCEPTING，在这个状态中对消息进行了一些简单的处理，然后将state的值置为STATE_ACCEPTING_WAIT_BANNER_ADDR。类似于TCP的三次握手过程。每个接收状态下消息模块都会对消息进行一些简单的处理操作，比如Open消息，读取其中的头部、中间部分、数据部分，最后读取数据等，下面主要介绍消息接收状态(state)的转换过程，由于每个状态下对消息进行了相应的处理，直到STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH状态时消息接收完毕。

### AsyncConnection的状态转换

<img src="D:\typora\workspace\img\AsyncConnection_stat.jpg" alt="AsyncConnection_stat" style="zoom:200%;" />


### 消息接收转换

![消息接收状态转换图.jpg](..\img\消息接收状态转换图.jpg) 

## 消息处理

![image-20210722162025422](C:\Users\13729\AppData\Roaming\Typora\typora-user-images\image-20210722162025422.png)

经过上述的一系列状态变换，消息通信接收端读出消息中的包含信息，单读出的数据大部分都在bufferlist中，如果将bufferlist数据发出去，Dispatch无法处理这些数据，因此，需要一个将bufferlist中的数据封装成Message的过程。

在STATE_OPEN_MESSAGE_READ_FOOTER_AND_DISPATCH状态中，首先从数据中读出尾部消息，在通过调用Message *decode_message()来封装成消息。封装完消息成后，调用Message::set_connection()将当前的连接添加到消息的连接中，然后执行Messenger::ms_fast_preprocess()对消息的分发进行一个预处理，具体执行时注册的Dispatcher来操作的，比如OSD。

预处理完成以后对当前的消息进行一个判断，判断是否是delay状态，如果是，则将消息添加到delay_state->queue()。如果需要快速派送，调用Messenger::ms_fast_dispatch()，从fast_dispatchers链表中选择注册的Dispatcher，对消息进行快速派发。如果都不是，则将消息放到dispatch_queue->queue()中，Messenger从dispatchers链表中选择注册的Dispatcher对消息进行普通派发。
