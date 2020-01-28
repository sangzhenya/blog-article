## Netty EventLoop

NioEventLoop 的继承关系图如下：

![](http://img.programya.com/20200128220652.png)

首先看一下 SingleThreadEventLoop 类的 execute 方法：

```java
public void execute(Runnable task) {
  ObjectUtil.checkNotNull(task, "task");
  execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}
private void execute(Runnable task, boolean immediate) {
  boolean inEventLoop = inEventLoop();
  // 将 task 添加到队列中
  addTask(task);
  if (!inEventLoop) {
    // 如果该 EventLoop 线程不是当前线程则尝试启动线程
    startThread();
    if (isShutdown()) {
      boolean reject = false;
      try {
        // 如果启动失败，从 task 队列中移除当前 task
        if (removeTask(task)) {
          reject = true;
        }
      } catch (UnsupportedOperationException e) {
        // The task queue does not support removal so the best thing we can do is to just move on and
        // hope we will be able to pick-up the task before its completely terminated.
        // In worst case we will log on termination.
      }
      if (reject) {
        reject();
      }
    }
  }

  if (!addTaskWakesUp && immediate) {
    // Use offer as we actually only need this to unblock the thread and if offer fails we do not care as there
    // is already something in the queue.
    wakeup(inEventLoop);
  }
}
```

```java
private void startThread() {
  if (state == ST_NOT_STARTED) {
    if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
      boolean success = false;
      try {
        // 确实需要 start 的时候才 start
        doStartThread();
        success = true;
      } finally {
        if (!success) {
          STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
        }
      }
    }
  }
}
private void doStartThread() {
  assert thread == null;
  executor.execute(new Runnable() {
    @Override
    public void run() {
      // 获取当前线程
      thread = Thread.currentThread();
      if (interrupted) {
        thread.interrupt();
      }

      boolean success = false;
      // 更新时间
      updateLastExecutionTime();
      try {
        // 调用 run 方法启动 EventLoop
        SingleThreadEventExecutor.this.run();
        success = true;
      } catch (Throwable t) {
        logger.warn("Unexpected exception from an event executor: ", t);
      } finally {
        for (;;) {
          int oldState = state;
          if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
            break;
          }
        }

        // Check if confirmShutdown() was called at the end of the loop.
        if (success && gracefulShutdownStartTime == 0) {
          if (logger.isErrorEnabled()) {
            logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                         SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                         "be called before run() implementation terminates.");
          }
        }

        try {
          // Run all remaining tasks and shutdown hooks. At this point the event loop
          // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
          // graceful shutdown with quietPeriod.
          for (;;) {
            if (confirmShutdown()) {
              break;
            }
          }

          // Now we want to make sure no more tasks can be added from this point. This is
          // achieved by switching the state. Any new tasks beyond this point will be rejected.
          for (;;) {
            int oldState = state;
            if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
              SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
              break;
            }
          }

          // We have the final set of tasks in the queue now, no more can be added, run all remaining.
          // No need to loop here, this is the final pass.
          confirmShutdown();
        } finally {
          try {
            cleanup();
          } finally {
            // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
            // the future. The user may block on the future and once it unblocks the JVM may terminate
            // and start unloading classes.
            // See https://github.com/netty/netty/issues/6596.
            FastThreadLocal.removeAll();

            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
            threadLock.countDown();
            int numUserTasks = drainTasks();
            if (numUserTasks > 0 && logger.isWarnEnabled()) {
              logger.warn("An event executor terminated with " +
                          "non-empty task queue (" + numUserTasks + ')');
            }
            terminationFuture.setSuccess(null);
          }
        }
      }
    }
  });
}
```

```java
// NioEventLoop 的 run 方法：包括 select，processSelectedKeys，runAllTasks
protected void run() {
  int selectCnt = 0;
  for (;;) {
    try {
      int strategy;
      try {
        strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
        switch (strategy) {
          case SelectStrategy.CONTINUE:
            continue;

          case SelectStrategy.BUSY_WAIT:
            // fall-through to SELECT since the busy-wait is not supported with NIO

          case SelectStrategy.SELECT:
            long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
            if (curDeadlineNanos == -1L) {
              curDeadlineNanos = NONE; // nothing on the calendar
            }
            nextWakeupNanos.set(curDeadlineNanos);
            try {
              if (!hasTasks()) {
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

      selectCnt++;
      cancelledKeys = 0;
      needsToSelectAgain = false;
      final int ioRatio = this.ioRatio;
      boolean ranTasks;
      if (ioRatio == 100) {
        try {
          if (strategy > 0) {
            processSelectedKeys();
          }
        } finally {
          // Ensure we always run tasks.
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

调用 Selector 的select 方法默认阻塞 1秒，如果有定时任务，则在定时任务剩余的基础上加 0.5 进行阻塞。当执行 execute 方法的时候，也就是添加任务的时候，会唤醒 selector ，防止 selector 阻塞时间过长。