## Netty 心跳机制

Netty 提供了一个重要的服务心跳机制 heartbeat。通过心跳检查对方是否有效，这是 RPC 框架中必不可少的功能。接下来分析相关源码的实现。

Netty 提供了 IdleStateHandler, ReadTimeoutHandler, WriteTimeoutHandler 三个 Handler 检测连接的有效性。

1. IdleStateHandler：当连接的空闲时间（读/写）太长时，将会触发一个 IdleStateEvent 事件，然后可以通过 ChannelInboundHandler 中重写 userEventTrigged 方法处理该事件。
2. ReadTimeoutHandler 如果在指定时间没有发生读事件，抛出异常并自动关闭这个连接，可以在 exceptionCaught 方法中处理这个异常。
3. WriteTimeoutHandler 当一个写操作不能再一定的事件内完成时，抛出异常并关闭连接，统一可以在 exceptionCaught 方法中处理这个异常。

对于 IdleStateHandler 主要有以下属性：

```java
// 是否考虑出站较慢的情况，默认值是 false
private final boolean observeOutput;
// 读事件空闲时间，0 表示禁用
private final long readerIdleTimeNanos;
// 写事件空闲时间
private final long writerIdleTimeNanos;
// 读或写空闲时间
private final long allIdleTimeNanos;
// 0 - none, 1 - initialized, 2 - destroyed
private byte state; 
```

该类继承自 ChannelDuplexHandler，实现了 handlerAdded 方法如下：

```java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
  if (ctx.channel().isActive() && ctx.channel().isRegistered()) {
    // channelActive() event has been fired already, which means this.channelActive() will
    // not be invoked. We have to initialize here instead.
    initialize(ctx);
  } else {
    // channelActive() event has not been fired yet.  this.channelActive() will be invoked
    // and initialization will occur there.
  }
}
private void initialize(ChannelHandlerContext ctx) {
  // Avoid the case where destroy() is called before scheduling timeouts.
  // See: https://github.com/netty/netty/issues/143
  switch (state) {
    case 1:
    case 2:
      return;
  }

  state = 1;
  // 初始化监控数据属性
  initOutputChanged(ctx);

  // 当前系统的时间
  lastReadTime = lastWriteTime = ticksInNanos();
  if (readerIdleTimeNanos > 0) {
    // schedule 方法会调用 EventLoop 的 schedule 方法，将定时任务添加队列中
    readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                                 readerIdleTimeNanos, TimeUnit.NANOSECONDS);
  }
  if (writerIdleTimeNanos > 0) {
    writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                                 writerIdleTimeNanos, TimeUnit.NANOSECONDS);
  }
  if (allIdleTimeNanos > 0) {
    allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                              allIdleTimeNanos, TimeUnit.NANOSECONDS);
  }
}
```

```java
// ReaderIdleTimeoutTask
private final class ReaderIdleTimeoutTask extends AbstractIdleTask {
  ReaderIdleTimeoutTask(ChannelHandlerContext ctx) {
    super(ctx);
  }
  @Override
  protected void run(ChannelHandlerContext ctx) {
    long nextDelay = readerIdleTimeNanos;
    if (!reading) {
      // 获取两次读取的事件间隔，在 channelReadComplete 方法中会修改 lastReadTime 的值
      nextDelay -= ticksInNanos() - lastReadTime;
    }
    if (nextDelay <= 0) {
      // 新开一个 schedule
      readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);
      boolean first = firstReaderIdleEvent;
      firstReaderIdleEvent = false;
      try {
        // 创建一个 IdleEvent，并调用 channelIdle 方法，告知该 Channel 处于 Idle 状态
        IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
        channelIdle(ctx, event);
      } catch (Throwable t) {
        ctx.fireExceptionCaught(t);
      }
    } else {
      // 新开一个 schedule
      readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
  }
}
```

```java
protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
  // 触发  UserEventTrigger 
  ctx.fireUserEventTriggered(evt);
}
```

```java
public ChannelHandlerContext fireUserEventTriggered(final Object event) {
  // 调用 pipeline 中所有的 handler 处理事件
  invokeUserEventTriggered(findContextInbound(MASK_USER_EVENT_TRIGGERED), event);
  return this;
}
```

IdleStateHandler 可以实现心跳功能，当服务器和客户端没有任何读写交互时，并超过给定的事件，则会触发 handler 的 userEventTrigger 方法，用户可以在该方法中尝试向对方发送消息，如果发送失败，则关闭连接。IdleStateHandler 的实现基于 EventLoop 的定时任务，每次读写都会记录一个值，在定时任务运行的时候，通过继续当前时间和设置时间和上次事件发生事件的结果来判断是否 idle 了。内部有 3 个定时任务，分别对应读事件，写事件，读写事件。通常用户监听读写事件就足够了。

与此同时 IdleHandler 内部也考虑了一些极端的请求，客户端接收缓慢，一次接收数据的速到超过了设置的空闲时间。Netty 通过构造方法中的 observeOutput 属性决定是否对出站缓冲区的情况进行判断。如果出站缓慢，Netty 不认为是空闲也不会触发空闲事件。但是第一次无论如何都会触发的，因为第一次无法判断是出站缓慢还是空闲。当然如果出站缓慢可能造成 OOM 等更大的问题。所以当应用出现  OOM 之类的并且极少出现写空闲的时候需要特别注意是否是数据出站速度慢导致的。

此外 ReadTimeoutHandler 继承自 IdleStateHandler 当触发读写空闲事件的时候，就触发 ctx.fireExceptionCaught 然后关闭 Socket。

```java
protected void readTimedOut(ChannelHandlerContext ctx) throws Exception {
  if (!closed) {
    ctx.fireExceptionCaught(ReadTimeoutException.INSTANCE);
    ctx.close();
    closed = true;
  }
}
```

而 WriteTimeoutHandler 的实现不是基于 IdleStateHandler 的，其原理是当调用 write 方法的时候会创建一个定时 Task，Task 内容是根据传入的 Promis 的完成情况判断是否超出了读写事件。当定时任务根据指定时间开始运行，发现 promise 的 isDone 为 false 的时候表示没有写完即超时了，则抛出异常。当 write 方法完成后会打断定时任务。

```java
private void scheduleTimeout(final ChannelHandlerContext ctx, final ChannelPromise promise) {
  // 添加一个 write timeout 的 task
  final WriteTimeoutTask task = new WriteTimeoutTask(ctx, promise);
  task.scheduledFuture = ctx.executor().schedule(task, timeoutNanos, TimeUnit.NANOSECONDS);
  if (!task.scheduledFuture.isDone()) {
    addWriteTimeoutTask(task);
    promise.addListener(task);
  }
}
```

```java
private final class WriteTimeoutTask implements Runnable, ChannelFutureListener {

  private final ChannelHandlerContext ctx;
  private final ChannelPromise promise;

  // WriteTimeoutTask is also a node of a doubly-linked list
  WriteTimeoutTask prev;
  WriteTimeoutTask next;

  ScheduledFuture<?> scheduledFuture;

  WriteTimeoutTask(ChannelHandlerContext ctx, ChannelPromise promise) {
    this.ctx = ctx;
    this.promise = promise;
  }

  @Override
  public void run() {
    // 判断事件是否完成
    if (!promise.isDone()) {
      try {
        writeTimedOut(ctx);
      } catch (Throwable t) {
        ctx.fireExceptionCaught(t);
      }
    }
    removeWriteTimeoutTask(this);
  }

  @Override
  public void operationComplete(ChannelFuture future) throws Exception {
    // scheduledFuture has already be set when reaching here
    scheduledFuture.cancel(false);
    removeWriteTimeoutTask(this);
  }
}
```



