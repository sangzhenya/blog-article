---
title: "Netty 三大组件"
tags: ["Netty", "源码"]
categories: ["Netty"]
date: "2019-03-25T09:00:00+08:00"
---

## ChannelPipeline, ChannelHandler 和 ChannelHandlerContext

三者的关系是每当 ServerSocket 创建一个新的连接，就会创建一个 Socket 对应的就是目标客户端。每一个新创建的 Socket 都将会分配一个全新的 ChannelPipeline。每个 ChannelPipeline 包含多个 ChannelHandlerContext，其是一个双向链表，包装了通过 addLast 方法添加的 ChannelHandler。

![](http://img.programya.com/20200123231156.png)

ChannelSocket 和 ChannelPipeline 是一对一的关联关系，而 pipeline 内部的多个 Context 形成了链表，Context 只是对 Handler 的封装。当一个轻轻进来的时候，会进入 Socket 对应的 pipeline，并讲过 pipeline 中所有的 handler。

### ChannelPipeline

ChannelPipeline 继承了 inbound，outbound，iterable 等接口，表示其可以调用数据出入站的方法，同时也可以遍历内部的链表。

![](http://img.programya.com/20200123231335.png)

ChannelPipeline 本质上是一个 handler 的 双向链表，这个链表中即包含了 InboundHandler 又包含了 OutboundHandler，handler 用于处理或拦截入站和出站事件，pipeline 实现了过滤器的高级形式，以便用户控制事件如何处理，handler 在 pipeline 中如何交互。IO 事件在处理的时候会调用  ChannelHandlerContext 的 fireChannelRead 方法转发给其最近的处理程序。

```java
public ChannelHandlerContext fireChannelRead(final Object msg) {
  // 使用找到 context 进行读取操作
  invokeChannelRead(findContextInbound(MASK_CHANNEL_READ), msg);
  return this;
}
private AbstractChannelHandlerContext findContextInbound(int mask) {
  AbstractChannelHandlerContext ctx = this;
  do {
    // 遍历找到可以处理当前 msg 的 context
    ctx = ctx.next;
  } while ((ctx.executionMask & mask) == 0);
  return ctx;
}
// 与 findContextInbound 对应的有一个是 findContextOutbound
// 区别是前者从前向后找，后者从后向前找
private AbstractChannelHandlerContext findContextOutbound(int mask) {
  AbstractChannelHandlerContext ctx = this;
  do {
    ctx = ctx.prev;
  } while ((ctx.executionMask & mask) == 0);
  return ctx;
}
```

![](http://img.programya.com/20200119232204.png)

入站事件是由入站 Handler 从下而上的方向处理，入站 Handler 通常由底层的 IO 线程生成入站数据，入站数据通常从入 SocketChannel.read(ByetBuffer) 获取。

通常一个 pipeline 有多个 handler，例如典型服务器在每个 pipeline 中都会有一个协议解码器将二进制转换为 Java 对象，协议编码器将 Java 对象转换为 二进制数据，业务处理 Handler 执行具体的业务逻辑。业务程序不应该将线程阻塞，这样会影响 IO 的速度进而影响整个 Netty 的性能。如果业务程序很快可以放到 IO 线程中，否则就需要异步执行。或在 Handler 中添加一个线程池。例如 下面这个任务执行的时候就不会阻塞 IO 线程，执行的线程来自于 Group 线程池。

`pipeline.addLast(group, "handler", new MyCustomizedHandler);`

### ChannelHandler

ChannelHandler 的作用就是处理 IO 事件或拦截 IO 事件，并将其转发给下一个处理的 ChannelHandler，Handler 处理事件时分入站和出站，两个方向的操作是不同的，因此 Netty 定义了两个子接口继承 ChannelHandler 分别是 `ChannelInboundHandler` 和 `ChannelOutboundHandler` 。主要分方法有如下：

```java
// 添加了一个 Handler 之后调用
void handlerAdded(ChannelHandlerContext ctx);
// 删除了一个 Handler 之后调用
void handlerRemoved(ChannelHandlerContext ctx);
/**** InboundHandler****/
// Channel 处于 Active 状态的时候调用
void channelActive(ChannelHandlerContext ctx);
// Channel 处于 Inactive 状态的时候调用
void channelInactive(ChannelHandlerContext ctx);
// 从 Channel 读取数据的时候被调用
void channelRead(ChannelHandlerContext ctx, Object msg);
/**** OutboundHandler ****/
// 当请求将 Channel 绑定到本地地址时被调用
void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise);
// 当请求关闭 Channel 时被调用
void close(ChannelHandlerContext ctx, ChannelPromise promise);
```

此外还有一个 ChannelDuplexHandler 类既可以处理 Inbound 又可以处理 Outbound。

![](http://img.programya.com/20200118084724.png)



### ChannelHandlerContext 

![](http://img.programya.com/20200124153535.png)

ChannelHandlerContext 继承了 ChannelOutboundInvoker 和 ChannelInboundInvoker 的接口。这两个 Invoker 就是针对入站或出站方法的，就是在入站和出站的 handler 的外层再包装一层，达到在方法前后连接并做一些特殊的操作的目的。当然其不仅仅继承了他们两个方法，同时也定义了一些自己的方法。这些方法能够获取 Context 上下文环境中的对应的 channel, executor, handler, pipeline, 内存分配器，关联的 handler 是否被删除等信息。Context  就是包装了 handler 相关的一切，以便 Context 在 pipeline 方便的操作 handler。

### ChannelPipeline, ChannelHandler, ChannelHandlerContext 创建过程

任何一个 ChannelSocket 创建的同时都会创建一个 pipeline。当用户或系统内部调用 pipeline 的 add 方法添加 handler 时，都会创建包装这个 handler 的context，这些 context 在 pipeline 中组成了双向链表。

在创建 SocketChannel 的时候会去创建一个 pipeline

```java
protected AbstractChannel(Channel parent) {
  this.parent = parent;
  id = newId();
  unsafe = newUnsafe();
  // 创建 pipeline
  pipeline = newChannelPipeline();
}
```

```java
protected DefaultChannelPipeline newChannelPipeline() {
  // 创建一个 DefaultChannelPipeline
  return new DefaultChannelPipeline(this);
}
```

```java
protected DefaultChannelPipeline(Channel channel) {
  this.channel = ObjectUtil.checkNotNull(channel, "channel");
  // 创建 future 和 promise 用于异步回调使用
  succeededFuture = new SucceededChannelFuture(channel, null);
  voidPromise =  new VoidChannelPromise(channel, true);

  // 创建 inbound 类型的 tailContext
  tail = new TailContext(this);
  // 创建既是 inbound 类型又是 outbound 类型的 headContext
  head = new HeadContext(this);

  // 将两个 context 互相连接形成双向链表
  head.next = tail;
  tail.prev = head;
}
```

在调用 add 添加处理器的时候会创建 context 如下：

```java
// 参数值为是 线程池，handler 使我们或者系统传入的 handler，
// Netty 为了防止多个线程导致安全问题，同步了这段代码
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
  final AbstractChannelHandlerContext newCtx;
  synchronized (this) {
    // 检查当前 handler 实例是否共享，如如果不是且被其他 pipeline 使用了，则抛出异常
    checkMultiplicity(handler);
		// 将 handler 包装成一个 context
    newCtx = newContext(group, filterName(name, handler), handler);

    // 将包装后的 context 添加到链表中
    addLast0(newCtx);

    // 如果这个 Channel 还没有注册到 selector 上，就将这个 context添加到 pipeline 的待办任务中
    // 当注册之后会调用 callHandlerAdded0 方法，默认什么都不做，用户可以实现这个方法
    if (!registered) {
      newCtx.setAddPending();
      callHandlerCallbackLater(newCtx, true);
      return this;
    }

    EventExecutor executor = newCtx.executor();
    if (!executor.inEventLoop()) {
      callHandlerAddedInEventLoop(newCtx, executor);
      return this;
    }
  }
  callHandlerAdded0(newCtx);
  return this;
}
```

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
  // 将 handler 包装成 context 对象
  return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}
```

```java
// 添加到双向链表中
private void addLast0(AbstractChannelHandlerContext newCtx) {
  AbstractChannelHandlerContext prev = tail.prev;
  newCtx.prev = prev;
  newCtx.next = tail;
  prev.next = newCtx;
  tail.prev = newCtx;
}
```

### ChannelPipeline 调度 handler 流程

首先当一个请求进来的时候，会第一个调用 pipeline 的相关方法，如果是入站事件这些方法由 fire 开头，表示开始管道的流动，让后面的 handler 继续处理。 DefaultChannelPipeline 中的一个 inbound 方法如下：

```java
public final ChannelPipeline fireChannelRead(Object msg) {
  // 从 head 开始处理
  AbstractChannelHandlerContext.invokeChannelRead(head, msg);
  return this;
}
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
  final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
  EventExecutor executor = next.executor();
  if (executor.inEventLoop()) {
    // 如果是当前的 EventLoop group 则直接调用 ChannelRead 方法
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
private void invokeChannelRead(Object msg) {
  if (invokeHandler()) {
    try {
      // 如果是 inbound 则直接调用 channelRead 方法
      ((ChannelInboundHandler) handler()).channelRead(this, msg);
    } catch (Throwable t) {
      notifyHandlerException(t);
    }
  } else {
    // 否则找下一个 handler
    fireChannelRead(msg);
  }
}
```

fire 这些都是 inbound 的方法，从 head 开始处理，这些静态方法则会调用 ChannelInboundInvoker 接口的方法，然后调用 handler 的真正方法。对于 outbound 的方法，则是从 tail 开始处理。

入站是 head 开始，出站时 tail 开始，因为出站时从内部向外部写，从 tail 开始能够让前面的 handler 进行处理，防止 handler 遗漏；反之入站是从 head 往内部输入，让后面的 handler 能够处理这些数据。虽然 head 也实现了 outbound 接口，但是不从 head 开始执行出站任务。