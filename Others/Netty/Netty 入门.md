---
title: "Netty 入门"
tags: ["Netty", "源码"]
categories: ["Netty"]
date: "2019-03-23T09:00:00+08:00"
---

Netty 是由 JBOSS 提供的一个 Java 开源框架，现在是 GitHub 上的独立项目。其是一个异步的，基于事件驱动的网络应用框架，也来快速开发高性能，高可用的网络 IO 程序。主要针对在 TCP 协议下，面向 Clients 端的高并发应用，或者在 P2P 场景下的大量数据持续传输的应用。本质上还是一个 NIO 框架，适用于服务器通讯相关的多种应用场景。

在分布式系统中各个节点需要远程服务调用，高性能的 RPC 框架则是必要的，Netty 作为异步高性能的通讯框架，往往作为基础通讯组件被这些 RPC 框架使用，例如 Dubbo。

### 基本概念

Netty 架构图如下：

![Netty 架构图](http://img.programya.com/20200115230555.png)

Netty 主要的优点如下：

1. 设计优雅，适用于各种传输类型的统一 API 阻塞和非阻塞 Socket；基于灵活且可扩展的事件模型。可以清晰的分离关注点；高度可定制的线程模型 - 单线程，一个或多个线程池。
2. 使用方便，文档详细，仅依赖 JDK 1.6。
3. 高性能、吞吐量更高；延迟更低，资源消耗少，最小化不必要的内存复制。
4. 安全：完整的 SSL/TLS 和 StartTLS 支持。
5. 社区活跃，迭代快速。

Netty 主要基于 Reactor 多线程模型做了一定的改进，其中主从 Reactor 多线程模型有多个 Reactor。

![Netty 线程模式](http://img.programya.com/20200116221706.png)

Netty 抽象出两组线程池 BossGroup 专门负责客户端的连接，WorkGroup 专门负责网络的读写。BossGroup 和 WorkGroup 类型都是 NioEventLoopGroup，NioEventLoopGroup 相当于一个一个事件循环组，这个组中包含多个事件循环，每个事件循环都是一个 NioEventLoop。NioEventLoop 表示一个不断循环的执行处理任务的线程，内部采用串行化设计，负责消息的读取，解码，处理，编码，发送等，每个 NIOEventLoop 都有一个 Selector 由于监听绑定在其上的 socket 的网络通讯。NIOEventLoopGroup 可以有多个线程，即可以包含多个 NIOEventLoop。每个 BoosEventLoop 循环步骤有 3 步：

1. 轮询 accept 事件
2. 处理 accept 事件，与 Client 建立连接，生成 NioSocketChannel 并将其注册到 work NioEventLoop 上的 selector。
3. 处理任务队列的任务，即 runAllTasks

每个 Work NioEventLoop 循环执行步骤：

1. 轮询 read, write 事件
2. 处理 io 事件，即 read 和 write 事件，在对应的 NioSocketChannel 处理。
3. 处理任务队列的任务，即 runAllTasks。

每个 Worker NIOEventLoop 处理业务时，会使用 pipeline， 其保护了 Channel 即 通过pipeline 可以获取到对应的 Channel，其中维护了很多的处理器。

### TaskQueue

TaskQueue 中的 Task 有 3 种典型使用场景：

1. 用户程序自定义的普通任务
2. 用户自定义定时任务
3. 非当前 Reactor 线程调用 Channel 的这种方法

```java
ctx.channel().eventLoop().execute(() -> {
  try {
    Thread.sleep(10 * 1000);
    ctx.writeAndFlush(Unpooled.copiedBuffer("Hello, client, 2", CharsetUtil.UTF_8));
  } catch (Exception e) {
    e.printStackTrace();
  }
});

ctx.channel().eventLoop().schedule(() -> {
  try {
    Thread.sleep(10 * 1000);
    ctx.writeAndFlush(Unpooled.copiedBuffer("Hello, client, 2", CharsetUtil.UTF_8));
  } catch (Exception e) {
    e.printStackTrace();
  }
}, 20, TimeUnit.SECONDS);
```

### 异步模型

当一个异步调用过程调用发出后，调用者不能立刻得到结果，实际的处理这个调用的组件在完成之后，通过状态，通知和回调来通知调用者。Netty 中的 IO 操作是异步的 包括 Bind Write Connect 等操作会简单的返回一个 ChannelFuture。通过 Future Listener 机制，用户可以方便的主动获取或者通过通知机制获得 IO 操作结果。Netty 的异步模型也是建立在 future和 callback 之上的。Future 的核心思想就是假设一个方法计算过程可能比较耗时，可以通过都在调用该方法的时候立马返回一个 Future，后续可以通过 Future 去监控方法的处理过程。 

![工作原理](http://img.programya.com/20200117221417.png)

在使用 Netty 进行编程的时候，拦截操作和转换出入站数据只需要提供 callback 或者利用 future 即可。这使得链式操作简单、高效并有利于编写可重用的、通用的代码。Netty 框架的目的就是让业务逻辑从网络基础编码中分离出来，进而更高效的开发。

Future - Listener 机制：对 Future 对象刚刚创建时，处于非完成状态，调用者可以通过返回的 ChannelFuture 来获取操作执行的状态，注册监听函数执行完成的操作。常用的操作有：

```
isDone: 判断当前操作是否完成
isSuccess: 判断当前操作是否成功
getCause: 判断已完成的当前操作的失败的原因
isCancelled: 判断已完成的当前操作是否被取消
addListener: 注册监听器，当操作已完成的时候，将会通知指监听器；
如果 Future 对象已完成则通知指定的监听器
```

