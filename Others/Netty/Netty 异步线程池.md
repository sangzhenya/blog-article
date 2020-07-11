---
title: "Netty 异步线程池"
tags: ["Netty", "源码"]
categories: ["Netty"]
date: "2019-03-27T09:00:00+08:00"
---

在 Netty 中做耗时的，不可预料的操作，比如数据库，网络请求，会严重影响 Netty 对 Socket 的处理速度。解决方案则是将任务添加到异步线程池中，主要有两种方式：

1. handler 中加入线程池
2. context 中加入线程池

### Handler 中加入线程池

在 Handler 中添加一个 EventExecutorGroup，需要创建线程的时候使用 group.submit 创建线程，整个流程则变成如下：

![](http://img.programya.com/20200129085852.png)

当 IO 线程轮询到一个 Socket 事件，然后 IO 线程开始处理，当遇到耗时的 handler 的时候会将耗时的任务交给 业务线程池处理。当耗时的任务执行完毕再执行 pipeline write 方法的时候会将任务交给这个任务的 IO 线程。

看一下 AbstractChannelHandlerContext 的 write 方法

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
  ObjectUtil.checkNotNull(msg, "msg");
  try {
    if (isNotValidPromise(promise, true)) {
      ReferenceCountUtil.release(msg);
      // cancelled
      return;
    }
  } catch (RuntimeException e) {
    ReferenceCountUtil.release(msg);
    throw e;
  }

  final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                                                                 (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
  final Object m = pipeline.touch(msg, next);
  EventExecutor executor = next.executor();
  if (executor.inEventLoop()) {
    // 如果在当前线程则执行 write 或 writeAndFlush
    if (flush) {
      next.invokeWriteAndFlush(m, promise);
    } else {
      next.invokeWrite(m, promise);
    }
  } else {
    // 如果不在当前线程，则就当前任务封装成 task 然后放入 mpsc 队列中，
    // 等待 IO 任务执行完毕后执行队列中的任务
    final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
    if (!safeExecute(executor, task, promise, m, !flush)) {
      // We failed to submit the WriteTask. We need to cancel it so we decrement the pending bytes
      // and put it back in the Recycler for re-use later.
      //
      // See https://github.com/netty/netty/issues/8343.
      task.cancel();
    }
  }
}
```

### Context 中加入线程池

在添加 Handler 的类中添加一个 EventExecutorGroup，在添加 Handler 到 pipeline 中的时候使用 `p.addLast(group, serverHandler);` 进行添加。Handler 中代码就能使用普通的方式处理耗时业务了。当在调用 addLast 方法的添加线程池后，handler 优先使用这个线程池，如果不添加在使用 IO 线程。例如 AbstractChannelHandlerContext  的 invokeChannelRead 方法如下：

```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
  final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
  EventExecutor executor = next.executor();
  if (executor.inEventLoop()) {
    next.invokeChannelRead(m);
  } else {
    // 如果不是当前线程则使用 异步任务的方式执行
    executor.execute(new Runnable() {
      @Override
      public void run() {
        next.invokeChannelRead(m);
      }
    });
  }
}
```

### 两种方式的比较

第一种方式是在 Handler  中添加异步，更加自由，比如如果需要访问数据库，就可以异步，其他不需要异步的可以直接执行。此外异步会拖长接口响应的事件，因为需要将任务放入到 mpscTask 中，如果 IO 时间很短，task 很多，可能一个循环下去，都没有事件执行整个 task，导致响应时间过慢。

第二种方式是 Netty 标准方式，但是这么做会导致这个 handler 都交给业务线程池。不论是否耗时都加入到队列中。不够灵活。