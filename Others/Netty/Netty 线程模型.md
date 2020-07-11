---
title: "Netty 线程模型"
tags: ["Netty", "源码"]
categories: ["Netty"]
date: "2019-03-17T09:00:00+08:00"
---

目前主要存在两种线程模型：传统阻塞 IO 服务模型和 Reactor 模式。其中 Reactor 模式根据处理资源池线程的数量不同有三种典型的实现：

1. 单 Reactor 单线程
2. 单 Reactor 多线程
3. 主从 Reactor 多线程

Netty 是根据主从 Reactor 多线程模型做了一点的改进，其中的主从Reactor 多线程模型有多个 Reactor。

##### 传统阻塞 IO 服务模型

![传统阻塞 IO 服务模型](https://i.loli.net/2020/05/02/UxSauo6IWp52HiD.png)

黄色表示对象，蓝色表示线程，白色表示方法 API

该模型采用阻塞 IO 模式获取输入的数据；每个连接都需要独立的线程完成数据的输入，业务处理，数据返回；

问题是当并发数量很大的时候会创建大量的线程进而占用很多的系统资源；另外就是连接创建后，如果当前线程临时没有数据可以读且，那么线程会阻塞在 Read 操作，进而造成线程资源浪费。

##### Reactor 模式

![Reactor 模式](https://i.loli.net/2020/05/02/nSti95sL12uWKrO.png)

针对于 传统模式的缺点解决方法如下：

1. 基于 IO 服用模型，多个连接共用一个阻塞对象，应用程序只需要一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据时操作系统通知应用程序，线程从阻塞状态返回开始进行业务处理。
2. 基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理分配给线程进行处理，一个线程可以处理多个连接的业务。

Reactor 模式是通过一个或多个输入同时传递给服务器的模式，基于事件驱动；服务器程序处理穿度的多个请求并将它们同步分派到相应的处理线程，因此 Reactor 模式也叫 Dispatcher 模式；Reator 模式使用 IO 复用监听事件，收到事件后分发给某个线程，这是网络服务器高并发处理的关键。

主要优点是响应快，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的；可以最大程度避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；扩展性好，可以方便的通过增加 Reactor 实例来充分利用 CPU 资源；复用性好，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性。

Reactor 模式核心的组成：

1. Reactor：Reactor 在一个单独的线程中运行，负责监听和事件分发，分发给适当的处理程序对 IO 事件做出反应。
2. Handlers：处理程序执行 IO 事件要完成的实际事件。Reactor 通过调度适当的 处理程序来相应 IO 事件，处理程序执行非阻塞操作。

**单 Reactor 单线程**

![单 Reactor 单线程](https://i.loli.net/2020/05/02/c8kAv4m1xaTHt5B.png)

Select 是 IO 复用模型介绍的标准网络编程 API，可以实现应用程序通过一个阻塞对象监听多路连接请求。Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发。如果是建立连接的请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理。如欧冠不是连接事件，则 Reactor 会分发调用相应的 Handler 来响应。Handler 会完成 Read，业务处理，Send 的完整业务流程。

优点是模型简单，没有多线程，进程通讯，竞争的问题，因为都在一个线程中。

缺点是性能问题，只有一个线程，无法实现发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。可靠性问题，线程可能意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收或者处理外部消息，造成节点故障。

适用场景是客户端数量有限，业务处理非常快速，比如 Redis 在业务处理的时间复杂度 O(1) 的情况。

**单 Reactor 多线程**

![单 Reactor 多线程](https://i.loli.net/2020/05/02/IaEzQtAVCYHDdsv.png)

Reactor 对象通过 select 监控客户端请求事件，收到事件后，通过 Dispatch 进行分发。如果是连接请求则由 Acceptor 通过 accept 完成连接请求，然后创建一个 Handler 对象处理完成连接后的各种事件。如果不是连接请求，则由 Reactor 分发调用连接对应的 Handler 来处理。Handler 只负责响应事件，不做具体的业务处理，通过 Read 读取数据后会分发后面的 worker 线程池的某个线程处理业务。worker 线程池会分配独立线程完成真正的业务，并将结果返回给 handler，handler 收到响应后通过  send 将结果返回给 Client。

优点可以充分利用多核 CPU 的处理能力

缺点多线程数据共享和访问比较复杂，Reactor 处理所有的事件的监听和响应，在单线程运行，在高并发的场景容易出现性能瓶颈。

**主从 Reactor 多线程**

![主从 Reactor 多线程](https://i.loli.net/2020/05/02/ICkZxeu4JBM867A.png)

针对单 Reactor 多线程模型中的问题，可以让 Reactor 在多线程中运行。Reactor 主线程 MainReactor 对象通过 Select 监听连接事件，收到事件后，通过 Acceptor 处理连接事件。当 Acceptor 处理连接事件后，MainReactor 将连接分配给 SubReactor ，SubReactor 将连接加入到连接队列进行监听并创建 handler 进行各种事件处理。当有新事件发生时，SubReactor 就会调用对应的 Handler 处理，handler 通过 read 读取数据，分发给后面的 worker 线程处理。worker 线程池分配独立的 worker 线程进行业务处理，并返回结果。handler 收到响应后再通过 send 将结果返回给 Client，Reactor 主线程可以对应多个 Reactor 子线程，即 MainReactor 可以关联多个 SubReactor。

优点父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。缺点编程复杂度较高。

这种模式在许多项目中广泛应用，包括 Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持。