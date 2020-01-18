## Netty 核心

### Bootstrap 和 ServerBootstrap

Bootstrap 即导引，一个 Netty 应用通常由一个 Boostrap 开始，主要作用是配置整个 Netty 程序，串联各个组件，Netty 中的 Bootstrap 类是客户端程序的启动引导类，ServerBootstrap 是服务端启动引导类。常用方法如下：

```java
// 服务端用来设置两个 Event Group
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup);
// 客户端用来设置一个 Event Group
public ServerBootstrap group(EventLoopGroup group);
// 用了设置一个服务器端的 Channel 实现
public B channel(Class<? extends C> channelClass);
// 用来个 ServerChannel 添加配置
public <T> B option(ChannelOption<T> option, T value);
// 用来给接收到的 Channel 添加配置
public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value);
// 用来设置业务处理类
final ChannelHandler childHandler();
// 服务端设置绑定端口
public ChannelFuture bind(int inetPort);
// 客户端用于连接服务器
public ChannelFuture connect(String inetHost, int inetPort);
```

### Future 和 ChannelFuture

Netty 中的所有 IO 操作都是异步的，不能立刻得知消息是否被正确处理。但是可以过一会等其执行完成或者直接注册一个监听，具体是通过 Future 和 ChannelFutre 实现的，他们可以注册一个监听，在操作成功或者失败的时候自动触发注册的监听事件。常用方法有：

```java
// 返回当正在进行 IO 操作的 Channel
Channel channel();
// 等待异步操作执行完毕
ChannelFuture sync();
```

### Channel

Netty 网络通讯的组件，能够用于执行网络 IO 操作，通过 Channel 可以获得网络连接 Channel 的状态和网络连接的配置参数。其还提供一部网络 IO 操作（建立连接，绑定端口，读写等），调用返回一个 ChannelFuture  实例，通过注册监听到 ChannelFuture 上，可以在 IO 操作成功，失败或者取消的时候调用注册的监听器。Channel 还支持关联 IO 操作和对应处理程序。不同协议，不同阻塞类型的连接都有不同的 Channel 与之对应，常用的 Channel 类型如下：

```
NioSocketChannel：异步客户端 TCP Socket 连接
NioServerSocketChannel：异步服务器端 TCP Socket 连接
NioDatagramChannel：异步 UDP 连接
NioSctpChannel：异步客户端 Sctp 连接
NioSctpServerChannel：异步服务器端 Sctp 连接，涵盖 UDP/TCP/文件 IO
```

### Selector

Netty  基于 Selector 对象实现多路 IO 复用，通过 Selector 一个线程可以监听多个连接的 Channel 事件，当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以自动不断的查询这些注册 Channel 是否已经有就绪的 IO 操作，可以使用一个线程高效的管理多个 Channel。

### ChannelHandler

ChannelHandler 是一个接口，处理 IO 事件或者拦截 IO 操作并将其转发到其 ChannelPipeline 中的下一个处理程序。其本有很多的子类实现，使用的时候可以继承其子类。主要入下图所示：

![ChannelHandler 及子类](http://img.programya.com/20200118084724.png)

```
ChannelInboundHandler: 处理入站 IO 事件
ChannelOutboundHandler: 处理出站 IO 事件
ChannelInboundHandlerAdapter: 处理入站 IO 事件
ChannelOutboundHandlerAdapter: 处理出站 IO 事件
ChannelDuplexHandler: 处理出入站事件
```

一般需要重写的方法如下：

```java
// 注册事件
public void channelRegistered(ChannelHandlerContext ctx);
// 取消注册事件
public void channelUnregistered(ChannelHandlerContext ctx);
// 就绪事件
public void channelActive(ChannelHandlerContext ctx);
public void channelInactive(ChannelHandlerContext ctx);
// 读取事件
public void channelRead(ChannelHandlerContext ctx, Object msg);
// 读取结束
public void channelReadComplete(ChannelHandlerContext ctx);
public void userEventTriggered(ChannelHandlerContext ctx, Object evt);
public void channelWritabilityChanged(ChannelHandlerContext ctx);
// 出现异常
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause);
```

### Pipeline 和 ChannelPipeline

ChannelPipeline 是一个 Handler 的集合，负责处理和连接 Inbound 或者 Outbound 的事件和操作，相当于一个贯穿 Netty 的链。其实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各自的 ChannelHandler 如何相互交互。

在 Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应，其组成关系如下：

![](http://img.programya.com/20200118092350.png)

一个 Channel 包含一个 ChannelPipeline，而 ChannelPipeline 中又维护了一个由 ChannelHandler 组成的双向链表，并且每个 ChannelHandlerContext 中又关联着 ChannelHandler。入站事件和出站事件在一个双向链表中，入站时间会从链表 header 向后传递到最后一个入站的 handler；出站事件会从链表 tail 传递到最前面一个出站的 handler。两种类型的 handler 互不干扰。常用方法如下：

```java
// 添加一个处理类在链表第一个位置
ChannelPipeline addFirst(ChannelHandler... handlers);
// 添加一个处理类在链表最后一个位置
ChannelPipeline addLast(ChannelHandler... handlers);
```

### ChannelHandlerContext

其中保存 Channel 相关的所有信息，绑定了对应的 Pipeline，同时关联一个 ChannelHandler 对象。常用方法如下：

```java
// 关闭 Channel
ChannelFuture close();
// 刷新
ChannelOutboundInvoker flush();
// 将数据写到 ChannelPipeline 中，当前 ChannelHandler 的一个 ChannelHandler 开始处理
ChannelFuture writeAndFlush(Object msg);
```

### ChannelOption

Netty 在创建 Channel 实例后一般都需要设置 ChannelOption 参数，常用的参数如下：

```
ChannelOption.SO_BACKLOG: 对应 TCP/IP 协议 listen 函数中的 backlog 函数，
用来初始化服务器可连接队列大小。服务端处理客户端连接请求的是顺序处理的，所以同一时间
职能处理一个客户端连接。多个客户端来的时候，服务端将不能处理的客户端连接放在队列中等待处理，
backlog 指定队列的大小。
ChannelOption.SO_KEEPALIVE: 一直保持连接活动状态。
```

### EventLoopGroup 和 NioEventLoopGroup

EventLoopGroup 是一组 EventLoop 的抽象，Netty 为了更好的利用多核 CPU 资源，一般会有多个 EventLoop 同时工作，每个 EventLoop 维护一个 Selector 实例。其提供了一个 next 接口，可以从组里按照一定的规则获取其中一个 EventLoop 来处理任。在 Netty 服务器端编程中，一般需要提供两个 EventLoopGroup 例如：BossEventLoopGroup 和 WorkerEventLoopGroup。

通常一个服务端口即一个 ServerSocketChannel 对应一个 Selector 和一个 EventLoop 线程。BossEventLoopGroup 负责客户端的连接并将 SocketChannel 交给 WorkerEventLoopGroup 来进行 IO 处理。

BossEventLoopGroup 通常是一个单线程的 EventLoop，EventLoop 维护者一个注册了 ServerSocketChannel 的 Selector 实例。BossEventLoop 不断轮询 Selector 将连接事件分离出来，通常是 OP_ACCEPT 事件，然后接收到的 SocketChannel 交给 WorkerEventLoopGroup ，其会由 next 选择其中的一个 EventLoop 了将这个 SocketChannel 注册到其维护的 Selector 并对其后续 IO 事件进行处理。

### Unpooled 

Netty 提供的专门用来操作缓冲区的工具类，常用方法有：

```java
// 通过给定的数据和字符编码返回一个 ByteBuf 对象
private static ByteBuf copiedBuffer(CharBuffer buffer, Charset charset);
```

创建 ByteBuf，该对象包含一个数组 arr，在 Netty 的 buffer 中不需要 flip 进行反转，底层维护了 readerIndex， writerIndex 和 capacity 将 buffer 分成三个区域： 0 到 readerIndex 已经读取的区域；readerIndex  到 writerIndex 可读区域；writerIndex 才 capacity 可写区域。