## Netty 源码

一个 Server 端的 Sample Code 如下：

```java
public final class EchoServer {
    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // 配置 SSL 的信息
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }

        // 配置 Server 的信息
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
              // ServerSocketChannel Handler
             .handler(new LoggingHandler(LogLevel.INFO))
              // SocketChannel Handler
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });

            // 绑定端口并等待完成
            ChannelFuture f = b.bind(PORT).sync();

            // 等待 Channel 关闭
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

在配置 Server 中两个是 EventLoopGroup 是核心对象，其中 bossGroup 用于接收 TCP 请求，然后转发给 workerGroup，workerGroup 会获取到真正的连接并和连接进行通讯，例如读写解编码。其中 EventLoopGroup 是事件循环组即线程组，含有多个 EventLoop，可以注册 Channel，用于在事件循环中进行选择。如下图所示：

![](http://img.programya.com/image-20200122005440647.png)

EventLoopGroup 创建的时候其父类 MultithreadEventLoopGroup 中的构造方法，如下：

```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
  // 这里可以看出如果未收到设置线程数量，则默认使用 DEFAULT_EVENT_LOOP_THREADS 的值
  // 而 DEFAULT_EVENT_LOOP_THREADS 的值取值逻辑是 Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", 
  // NettyRuntime.availableProcessors() * 2)); 即 CPU 核心数量的 * 2
  super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

然后上面会继续调用父类 MultithreadEventExecutorGroup  的构造方法如下：

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
  // 确保线程数大于 0
  if (nThreads <= 0) {
    throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
  }

  // 使用默认的线程工厂创建一个 ThreadPerTaskExecutor
  if (executor == null) {
    executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
  }

  // 根据线程数创建一个固定大小的 EventExecutor ，EventLoop 是其实现类
  children = new EventExecutor[nThreads];

  for (int i = 0; i < nThreads; i ++) {
    boolean success = false;
    try {
      // 使用 executor 和 args 创建一个 EventLoop 对象作为 Child
      children[i] = newChild(executor, args);
      success = true;
    } catch (Exception e) {
      // TODO: Think about if this is a good exception type
      throw new IllegalStateException("failed to create a child event loop", e);
    } finally {
      // 如果任何一个 Child 创建失败则关闭所有已经成功创建的 Child
      if (!success) {
        for (int j = 0; j < i; j ++) {
          children[j].shutdownGracefully();
        }

        for (int j = 0; j < i; j ++) {
          EventExecutor e = children[j];
          try {
            while (!e.isTerminated()) {
              e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
            }
          } catch (InterruptedException interrupted) {
            // Let the caller handle the interruption.
            Thread.currentThread().interrupt();
            break;
          }
        }
      }
    }
  }

  // 根据 children 的数量选择一个 chooser
  // Default implementation which uses simple round-robin to choose next
  chooser = chooserFactory.newChooser(children);

  // 创建 terminationListener 并逐个设置到 Children 上
  final FutureListener<Object> terminationListener = new FutureListener<Object>() {
    @Override
    public void operationComplete(Future<Object> future) throws Exception {
      if (terminatedChildren.incrementAndGet() == children.length) {
        terminationFuture.setSuccess(null);
      }
    }
  };

  for (EventExecutor e: children) {
    e.terminationFuture().addListener(terminationListener);
  }

  // 将 Children 放到链表中并设置为只读
  Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
  Collections.addAll(childrenSet, children);
  readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

Child 的创建方法如下：

```java
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
  	// 创建一个 NioEventLoop，基本上就是将传入的参数设置成为其全局属性值
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
```

NioEventLoop 和 NioEventLoopGroup 的类继承关系如下：

![](http://img.programya.com/image-20200122011533509.png)

除了两个 EventLoopGroup 外还有一个 ServerBootstrap 对象，其是一个引导类，用于启动服务器和引导整个程序的初始化，其和 ServerChannel 关联，ServerChannel 是 Channel 的子接口。

然后调用其 group 方法，传入两个 EventLoopGroup，第一个是 parentGroup 即 acceptor，第二个是 childGroup 即 client，都使用用于处理 ServerChannel 的事件和 IO。

然后调用 channel 方法设置需要初始化 Channel 实例的时候使用的子类：

```java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
            ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}
```

然后调用 option 方法设置一些属性相关的信息：

```java
public <T> B option(ChannelOption<T> option, T value) {
  ObjectUtil.checkNotNull(option, "option");
  if (value == null) {
    options.remove(option);
  } else {
    options.put(option, value);
  }
  return self();
}
```

然后调用 handler 方法设置用于处理 request 的 handler 信息：

```java
public B handler(ChannelHandler handler) {
    this.handler = ObjectUtil.checkNotNull(handler, "handler");
    return self();
}
```

然后调用 childHandler 方法设置用于处理 Channel 的request 的类。

```java
public ServerBootstrap childHandler(ChannelHandler childHandler) {
  this.childHandler = ObjectUtil.checkNotNull(childHandler, "childHandler");
  return this;
}
```

build 完成之后就可以调用 ServerBootstrap 的 bind 方法绑定一个端口了，绑定方法返回的是一个 ChannelFuture 对象。

```java
public ChannelFuture bind(int inetPort) {
  // 基于端口的信息创建一个 InetSocketAddress，用于 Netty 端口的绑定
  return bind(new InetSocketAddress(inetPort));
}
```

具体的绑定逻辑在 doBind 方法中

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    // 首先是初始化及注册
    final ChannelFuture regFuture = initAndRegister();
  	// 获取注册的 Channel
    final Channel channel = regFuture.channel();
  	// 如果异常则直接返回
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // 如果成功则继续处理
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
      	// 如果还没有处理完，则添加一个 listener 等处理完之后继续进行 bind 操作
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

```java
final ChannelFuture initAndRegister() {
  Channel channel = null;
  try {
    // 使用 ChannelFactory(基于 Channel 方法设置获取的工厂) 创建一个 Channel
    channel = channelFactory.newChannel();
    // 初始化 Channel
    init(channel);
  } catch (Throwable t) {
    if (channel != null) {
      // channel can be null if newChannel crashed (eg SocketException("too many open files"))
      channel.unsafe().closeForcibly();
      // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
      return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
    return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
  }

  // 将 Channel 注册到一个 EventLoop 上，返回一个 channelPromise 是 ChannelFuture 的子接口
  ChannelFuture regFuture = config().group().register(channel);
  // 如果注册失败了则关闭 Channel
  if (regFuture.cause() != null) {
    if (channel.isRegistered()) {
      channel.close();
    } else {
      channel.unsafe().closeForcibly();
    }
  }

  // If we are here and the promise is not failed, it's one of the following cases:
  // 1) If we attempted registration from the event loop, the registration has been completed at this point.
  //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
  // 2) If we attempted registration from the other thread, the registration request has been successfully
  //    added to the event loop's task queue for later execution.
  //    i.e. It's safe to attempt bind() or connect() now:
  //         because bind() or connect() will be executed *after* the scheduled registration task is executed
  //         because register(), bind(), and connect() are all bound to the same thread.

  return regFuture;
}
```

```java
void init(Channel channel) {
 	 // 设置 Channel 的一些配置信息
    setChannelOptions(channel, options0().entrySet().toArray(EMPTY_OPTION_ARRAY), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));
		// 获取 Channel 对应的 pipeline
    ChannelPipeline p = channel.pipeline();

    // 获取当前的 childGroup 和 childHandler 以及为 childChannel 设置的属性信息
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions =
            childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

  	// 在 pipeline 上注册 handler
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            // 获取 Channel 对应的 pipeline 并将 config 中配置的 handler 添加到 pipeline 上
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

          	// 根据 Channel， currentChildGroup，currentChildHandler 创建一个 ServerBootstrapAcceptor 
          	// 并将其添加到 pipeline 上。ServerBootstrapAcceptor 用于接收连接并将相关的事件注册到 childEventLoopGroup 中
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

```java
private static void doBind0(
  final ChannelFuture regFuture, final Channel channel,
  final SocketAddress localAddress, final ChannelPromise promise) {

  // 调用 Channel 的 bind 方法
  // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
  // the pipeline in its channelRegistered() implementation.
  channel.eventLoop().execute(new Runnable() {
    @Override
    public void run() {
      if (regFuture.isSuccess()) {
        channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
      } else {
        promise.setFailure(regFuture.cause());
      }
    }
  });
}
```

最终会调用到 AbstractChannelHandlerContext 中的 bind 方法：

```java
public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
  ObjectUtil.checkNotNull(localAddress, "localAddress");
  if (isNotValidPromise(promise, false)) {
    // cancelled
    return promise;
  }

  final AbstractChannelHandlerContext next = findContextOutbound(MASK_BIND);
 	// 获取 executor
  EventExecutor executor = next.executor();
  if (executor.inEventLoop()) {
    // invoke bind 方法
    next.invokeBind(localAddress, promise);
  } else {
    safeExecute(executor, new Runnable() {
      @Override
      public void run() {
        next.invokeBind(localAddress, promise);
      }
    }, promise, null, false);
  }
  return promise;
}
```

```java
private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
  if (invokeHandler()) {
    try {
      // 调用 ChannelOutboundHandler 的 bind 方法
      ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
    } catch (Throwable t) {
      notifyOutboundHandlerException(t, promise);
    }
  } else {
    bind(localAddress, promise);
  }
}
```

```java
public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
  synchronized (stateLock) {
    ensureOpen();
    if (localAddress != null)
      throw new AlreadyBoundException();
    InetSocketAddress isa = (local == null)
      ? new InetSocketAddress(0)
      : Net.checkAddress(local);
    SecurityManager sm = System.getSecurityManager();
    if (sm != null)
      sm.checkListen(isa.getPort());
    NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
    // 绑定地址
    Net.bind(fd, isa.getAddress(), isa.getPort());
    Net.listen(fd, backlog < 1 ? 50 : backlog);
    localAddress = Net.localAddress(fd);
  }
  return this;
}
```

initAndRegister 中新建 Channel 的主要流程是通过 NIO 的 SelectorProvider 和 openServerSocketChannel 方法得到 JDK 的 Channel，Netty 会包装 JDK 的Channel。创建一个唯一的 ChannelId，创建一个 NioMessageUnsafe 用于操作消息，创建了一个 DefaultChannelPipeline 管道，是一个双向链表，用于过滤所有进出的消息。创建一个 NioServerSocketChannelConfig 对象，用于对外展示一些配置。

```java
private static ServerSocketChannel newSocket(SelectorProvider provider) {
  try {
    // 通过 SelectorProvider 开启一个 ServerSocketChannel
    return provider.openServerSocketChannel();
  } catch (IOException e) {
    throw new ChannelException(
      "Failed to open a server socket.", e);
  }
}
```

```java
public ServerSocketChannel openServerSocketChannel() throws IOException {
 	// 创建 ServerSocketChannel
  return new ServerSocketChannelImpl(this);
}
```

```java
protected AbstractChannel(Channel parent) {
  this.parent = parent;
  // 创建 ID
  id = newId();
  // 创建 NioMessageUnsafe
  unsafe = newUnsafe();
  // 创建 DefaultChannelPipeline
  pipeline = newChannelPipeline();
}
```

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
  	// 创建 NioServerSocketChannelConfig
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

EventLoop 的 run 方法如下：

```java
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
                // 计算需要使用的处理策略，如果当前没有 Task 则返回 Select 策略 
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.BUSY_WAIT:
                case SelectStrategy.SELECT:
                    // 计算截止时间
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        curDeadlineNanos = NONE; // nothing on the calendar
                    }
                    // 设置标志位
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                          // 如果没有 task 则调用 select 方法
                          // 如果没有截止时间则会一直阻塞在这边
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        // This update is just to help block unnecessary selector wakeups
                        // so use of lazySet is ok (no race condition)
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

          	// 更新 select 的数量
            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
          // 获取 io 事件的比率（0-100 ，值越小表明花费在非 IO 事件的 Task 可以越多
          // 默认是 50，如果设置为 100 则 EventLoop 不会尝试平衡 IO 事件 Task 和 非 IO 事件 Task 比例
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                      // 处理  SelectKeys
                        processSelectedKeys();
                    }
                } finally {
                    // 确保会执行所有的 Task
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                ranTasks = runAllTasks(0); // This will run the minimum number of tasks
            }

          // 重置 Select Count
            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

里面包含了一个死循环，重要的方法有两个：processSelectedKeys  和 runAllTasks

其中 processSelectedKeys 如下：

```java
private void processSelectedKeys() {
  if (selectedKeys != null) {
   	// 处理 selectkeys
    processSelectedKeysOptimized();
  } else {
    processSelectedKeysPlain(selector.selectedKeys());
  }
}
private void processSelectedKeysOptimized() {
  // 遍历 Select keys 并逐个处理
  for (int i = 0; i < selectedKeys.size; ++i) {
    final SelectionKey k = selectedKeys.keys[i];
    selectedKeys.keys[i] = null;

    // 获取当前的附加信息
    final Object a = k.attachment();

    if (a instanceof AbstractNioChannel) {
      processSelectedKey(k, (AbstractNioChannel) a);
    } else {
      @SuppressWarnings("unchecked")
      NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
      processSelectedKey(k, task);
    }

    if (needsToSelectAgain) {
      selectedKeys.reset(i + 1);

      selectAgain();
      i = -1;
    }
  }
}
```

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
  final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
  if (!k.isValid()) {
    final EventLoop eventLoop;
    try {
      // 通过 Channel 获取当前的 EventLoop
      eventLoop = ch.eventLoop();
    } catch (Throwable ignored) {
      // If the channel implementation throws an exception because there is no event loop, we ignore this
      // because we are only trying to determine if ch is registered to this event loop and thus has authority
      // to close ch.
      return;
    }
    // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
    // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
    // still healthy and should not be closed.
    // See https://github.com/netty/netty/issues/5125
    if (eventLoop == this) {
      // close the channel if the key is not valid anymore
      unsafe.close(unsafe.voidPromise());
    }
    return;
  }

  try {
    // 获取 read 的 option
    int readyOps = k.readyOps();
    // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
    // the NIO JDK channel implementation may throw a NotYetConnectedException.
    if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
      // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
      // See https://github.com/netty/netty/issues/924
      int ops = k.interestOps();
      ops &= ~SelectionKey.OP_CONNECT;
      k.interestOps(ops);

      unsafe.finishConnect();
    }

    // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
    if ((readyOps & SelectionKey.OP_WRITE) != 0) {
      // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
      ch.unsafe().forceFlush();
    }

    // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
    // to a spin loop
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
      // OP_ACCEPT 事件，通过 unsafe 执行读操作
      unsafe.read();
    }
  } catch (CancelledKeyException ignored) {
    unsafe.close(unsafe.voidPromise());
  }
}
```

```java
public void read() {
  // 判断是当前的 EventLoop
  assert eventLoop().inEventLoop();
  // 获取 config、pipeline 等
  final ChannelConfig config = config();
  final ChannelPipeline pipeline = pipeline();
  final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
  allocHandle.reset(config);

  boolean closed = false;
  Throwable exception = null;
  try {
    try {
      do {
        // 执行去获取 Message
        int localRead = doReadMessages(readBuf);
        if (localRead == 0) {
          break;
        }
        if (localRead < 0) {
          closed = true;
          break;
        }

        // Increment the number of messages that have been read for the current read loop.
        allocHandle.incMessagesRead(localRead);
      } while (allocHandle.continueReading());
    } catch (Throwable t) {
      exception = t;
    }

    // 获取读取到的大小
    int size = readBuf.size();
    for (int i = 0; i < size; i ++) {
      readPending = false;
      // 循环调用 pipeline 触发 Channel read 的方法
      pipeline.fireChannelRead(readBuf.get(i));
    }
    readBuf.clear();
    // 读取完毕
    allocHandle.readComplete();
    pipeline.fireChannelReadComplete();

    if (exception != null) {
      closed = closeOnReadError(exception);
      pipeline.fireExceptionCaught(exception);
    }

    if (closed) {
      inputShutdown = true;
      if (isOpen()) {
        close(voidPromise());
      }
    }
  } finally {
    // Check if there is a readPending which was not processed yet.
    // This could be for two reasons:
    // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
    // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
    //
    // See https://github.com/netty/netty/issues/2254
    if (!readPending && !config.isAutoRead()) {
      removeReadOp();
    }
  }
}
```

```java
protected int doReadMessages(List<Object> buf) throws Exception {
  // 内部调用 ServerSocketChannel 的 accept 方法，获取一个 JDK 的SocketChannel
  SocketChannel ch = SocketUtils.accept(javaChannel());

  try {
    if (ch != null) {
      // 将 JDK 的 SocketChannel 封装成 NIOSocketChannel 然后添加到容器中
      buf.add(new NioSocketChannel(this, ch));
      return 1;
    }
  } catch (Throwable t) {
    logger.warn("Failed to create a new channel from an accepted socket.", t);

    try {
      ch.close();
    } catch (Throwable t2) {
      logger.warn("Failed to close a socket.", t2);
    }
  }

  return 0;
}
```

```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
  // 获取 msg
  final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
  // 获取当前 handler 的执行器
  EventExecutor executor = next.executor();
  if (executor.inEventLoop()) {
    // 调用 handler 的 invokeChannelRead（从 header 开始处理）
    next.invokeChannelRead(m);
  } else {
    executor.execute(new Runnable() {
      @Override
      public void run() {
        next.invokeChannelRead(m);
      }
    });
  }
}
```

```java
private void invokeChannelRead(Object msg) {
  if (invokeHandler()) {
    try {
      // 调用 handler 的 ChannelRead 方法
      ((ChannelInboundHandler) handler()).channelRead(this, msg);
    } catch (Throwable t) {
      notifyHandlerException(t);
    }
  } else {
    fireChannelRead(msg);
  }
}
public void channelRead(ChannelHandlerContext ctx, Object msg) {
  // 读取
  ctx.fireChannelRead(msg);
}
// 触发 ChannelRead 在其中会获取下一个 ctx 并开始读取 Channel 中的内容并做相应的处理
public ChannelHandlerContext fireChannelRead(final Object msg) {
  // 找到下一个 ctx 并开始读取
  invokeChannelRead(findContextInbound(MASK_CHANNEL_READ), msg);
  return this;
}
private AbstractChannelHandlerContext findContextInbound(int mask) {
  AbstractChannelHandlerContext ctx = this;
  do {
    ctx = ctx.next;
  } while ((ctx.executionMask & mask) == 0);
  return ctx;
}
// 具体读的方法
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
  if (logger.isEnabled(internalLevel)) {
    logger.log(internalLevel, format(ctx, "READ", msg));
  }
  ctx.fireChannelRead(msg);
}
```

```java
// ServerBootstrap.ServerBootstrapAcceptor#channelRead
public void channelRead(ChannelHandlerContext ctx, Object msg) {
  final Channel child = (Channel) msg;

  // 在客户端的 pipeline 上添加 child 的 hanlder 
  child.pipeline().addLast(childHandler);
	// 设置属性信息
  setChannelOptions(child, childOptions, logger);
  setAttributes(child, childAttrs);

  try {
    // 将客户端注册到 worker 线程池上
    childGroup.register(child).addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
        if (!future.isSuccess()) {
          forceClose(child, future.cause());
        }
      }
    });
  } catch (Throwable t) {
    forceClose(child, t);
  }
}
```

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
  // 一些校验判断
  ObjectUtil.checkNotNull(eventLoop, "eventLoop");
  if (isRegistered()) {
    promise.setFailure(new IllegalStateException("registered to an event loop already"));
    return;
  }
  if (!isCompatible(eventLoop)) {
    promise.setFailure(
      new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
    return;
  }

  // 设置当前的 EventLoop 
  AbstractChannel.this.eventLoop = eventLoop;

  if (eventLoop.inEventLoop()) {
    // 执行注册
    register0(promise);
  } else {
    try {
      eventLoop.execute(new Runnable() {
        @Override
        public void run() {
          // 异步的执行注册
          register0(promise);
        }
      });
    } catch (Throwable t) {
      logger.warn(
        "Force-closing a channel whose registration task was not accepted by an event loop: {}",
        AbstractChannel.this, t);
      closeForcibly();
      closeFuture.setClosed();
      safeSetFailure(promise, t);
    }
  }
}
```

```java
private void register0(ChannelPromise promise) {
  try {
    // check if the channel is still open as it could be closed in the mean time when the register
    // call was outside of the eventLoop
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
      return;
    }
    boolean firstRegistration = neverRegistered;
    // 执行 Register
    doRegister();
    neverRegistered = false;
    registered = true;

    // 通知 pipeline 中所有的 handler 有新的客户端连接
    pipeline.invokeHandlerAddedIfNeeded();
    safeSetSuccess(promise);
    // 通知pipeline 中所有的 handler 有 Channel 注册了
    // 和 read 的逻辑相似
    pipeline.fireChannelRegistered();
    // Only fire a channelActive if the channel has never been registered. This prevents firing
    // multiple channel actives if the channel is deregistered and re-registered.
    if (isActive()) {
      if (firstRegistration) {
        // 通知pipeline 中所有的 handler 有 Channel 活动了
        // 和 read 的逻辑相似
        pipeline.fireChannelActive();
      } else if (config().isAutoRead()) {
        // This channel was registered before and autoRead() is set. This means we need to begin read
        // again so that we process inbound data.
        //
        // See https://github.com/netty/netty/issues/4805
        beginRead();
      }
    }
  } catch (Throwable t) {
    // Close the channel directly to avoid FD leak.
    closeForcibly();
    closeFuture.setClosed();
    safeSetFailure(promise, t);
  }
}
```

```java
protected void doRegister() throws Exception {
  boolean selected = false;
  for (;;) {
    try {
      // 调用 JDK SelectableChannel 的注册方法
      selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
      return;
    } catch (CancelledKeyException e) {
      if (!selected) {
        // Force the Selector to select now as the "canceled" SelectionKey may still be
        // cached and not removed because no Select.select(..) operation was called yet.
        eventLoop().selectNow();
        selected = true;
      } else {
        // We forced a select operation on the selector before but the SelectionKey is still cached
        // for whatever reason. JDK bug ?
        throw e;
      }
    }
  }
}
```

在 readComplete 后的 pipeline 的 fireChannelReadComplete 方法中最终会调用到

```java
protected void doBeginRead() throws Exception {
  // Channel.read() or ChannelHandlerContext.read() was called
  final SelectionKey selectionKey = this.selectionKey;
  if (!selectionKey.isValid()) {
    return;
  }

  readPending = true;

  final int interestOps = selectionKey.interestOps();
  if ((interestOps & readInterestOp) == 0) {
    selectionKey.interestOps(interestOps | readInterestOp);
  }
}
```





其中  runAllTasks 方法如下

```java
protected boolean runAllTasks(long timeoutNanos) {
  // 获取 task
  fetchFromScheduledTaskQueue();
  Runnable task = pollTask();
  if (task == null) {
    afterRunningAllTasks();
    return false;
  }

  final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
  long runTasks = 0;
  long lastExecutionTime;
  for (;;) {
    // 执行 task
    safeExecute(task);

    runTasks ++;

   	// 避免超时
    if ((runTasks & 0x3F) == 0) {
      lastExecutionTime = ScheduledFutureTask.nanoTime();
      if (lastExecutionTime >= deadline) {
        break;
      }
    }

    task = pollTask();
    if (task == null) {
      lastExecutionTime = ScheduledFutureTask.nanoTime();
      break;
    }
  }

  afterRunningAllTasks();
  this.lastExecutionTime = lastExecutionTime;
  return true;
}
```

