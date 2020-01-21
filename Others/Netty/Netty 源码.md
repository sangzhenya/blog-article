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
             .handler(new LoggingHandler(LogLevel.INFO))
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

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
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

![image-20200122005440647](/Users/xinyue/Library/Application Support/typora-user-images/image-20200122005440647.png)

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

![image-20200122011533509](/Users/xinyue/Library/Application Support/typora-user-images/image-20200122011533509.png)

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

