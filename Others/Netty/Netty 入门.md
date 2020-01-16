## Netty 入门

Netty 是由 JBOSS 提供的一个 Java 开源框架，现在是 GitHub 上的独立项目。其是一个异步的，基于事件驱动的网络应用框架，也来快速开发高性能，高可用的网络 IO 程序。主要针对在 TCP 协议下，面向 Clients 端的高并发应用，或者在 P2P 场景下的大量数据持续传输的应用。本质上还是一个 NIO 框架，适用于服务器通讯相关的多种应用场景。

在分布式系统中各个节点需要远程服务调用，高性能的 RPC 框架则是必要的，Netty 作为异步高性能的通讯框架，往往作为基础通讯组件被这些 RPC 框架使用，例如 Dubbo。

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

Netty 抽象出两组线程池 BossGroup 专门负责客户端的连接，WorkGroup 专门负责网络的读写。BossGroup 和 WorkGroup 类型都是 NioEventLoopGroup，NioEventLoopGroup 相当于一个一个事件循环组，这个组中包含多个事件循环，每个事件循环都是一个 NIOEventLoop。NIOEventLoop 表示一个不断循环的执行处理任务的线程，每个 NIOEventLoop 都有一个 Selector 由于监听绑定在其上的 socket 的网络通讯。NIOEventLoopGroup 可以有多个线程，即可以包含多个 NIOEventLoop。每个 BoosEventLoop 循环步骤有 3 步：

1. 轮询 accept 事件
2. 处理 accept 事件，与 Client 建立连接，生成 NioSocketChannel 并将其注册到 work NioEventLoop 上的 selector。
3. 处理任务队列的任务，即 runAllTasks

每个 Work NioEventLoop 循环执行步骤：

1. 轮询 read, write 事件
2. 处理 io 事件，即 read 和 write 事件，在对应的 NioSocketChannel 处理。
3. 处理任务队列的任务，即 runAllTasks。

每个 Worker NIOEventLoop 处理业务时，会使用 pipeline， 其保护了 Channel 即 通过pipeline 可以获取到对应的 Channel，其中维护了很多的处理器。

