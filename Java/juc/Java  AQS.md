---
title: "Java AQS"
tags: ["java", "JUC"]
categories: ["Java"]
date: "2019-01-01T21:00:00+08:00"
# toc: true
---

众所周知 synchronized 是重量级锁，虽然在 JDK 1.6 后使用了轻量级锁，偏向锁，自适应自旋锁，锁粗化，锁消除等手段去去优化，但是其基于 JVM 的隐式获取和释放锁的方式导致缺少了获取锁和释放锁的可操作性。如果需要高效的实现锁并操作获取和释放锁的过程则需要使用显式锁 Lock，而其中 JUC 包下的 AQS 类是很多锁组件的基础。

AQS 即 `AbstractQueuedSynchronizer`, 是一个构建锁和同步队列的框架，底层是使用 CAS 保证操作的原子性，利用 FIFO 队列实现线程之间的锁竞争。 AQS 是 `ReentrantLock` ， `CountDownLatch`  和 `Semaphore` 等锁的底层实现机制，其将基础同步相关的抽象细节封装。使用 int 类型的 state 属性表示同步的状态，通过内置的基于  CLH  锁的 FIFO 队列来完成线程的管理。AQS 定义了使用者与锁交互的接口，将基础同步相关得抽象细节封装，大大降低了实现同步工具的工作。

AQS 依赖一个 FIFO 双向队列依赖来完成同步状态的管理，如果当前线程获取同步状态失败时，AQS 则会将当前线程已经等待的状态信息构造成一个节点并加入到同步队列中，同时会阻塞当前线程时。当同步状态释放的时候，会把首节点唤醒，使其再次尝试获取同步状态。队列中的每个线程都保存着线程的引用，状态信息，前驱节点，后继节点。


### LockSupport

AQS 中当需要阻塞或唤醒一个线程的时候，都会使用到 LockSupport 工具类了完成，其提供了一组公共静态的方法，完成最基本的线程阻塞和唤醒功能。

主要有三种的阻塞方法和一种唤醒方法。每种阻塞方法都有两种参数列表，即是否带 blocker, 其主要用来标识当前线程在等待的对象，主要用于问题排除和系统监控。

1. 简单的阻塞方法

   阻塞当前线程，直到当前线程被中断或者调用唤醒方法

   ```java
   public static void park() {
       UNSAFE.park(false, 0L);
   }
   public static void park(Object blocker) {
       Thread t = Thread.currentThread();
       setBlocker(t, blocker);
       UNSAFE.park(false, 0L);
       setBlocker(t, null);
   }
   ```

   

2. 带阻塞时长得阻塞方法

   阻塞当前线程，直到当前线程中断或者调用唤醒方法，或者超出指定得 nanos 数

   ```java
   public static void parkNanos(long nanos) {
       if (nanos > 0)
           UNSAFE.park(false, nanos);
   }
   public static void parkNanos(Object blocker, long nanos) {
       if (nanos > 0) {
           Thread t = Thread.currentThread();
           setBlocker(t, blocker);
           UNSAFE.park(false, nanos);
           setBlocker(t, null);
       }
   }
   ```

   

3. 带截止制时间的阻塞方法

   阻塞当前线程，直到当前线程中断或者调用唤醒方法，或者超过指定的 Deadline

   ```java
   public static void parkUntil(long deadline) {
       UNSAFE.park(true, deadline);
   }
   public static void parkUntil(Object blocker, long deadline) {
       Thread t = Thread.currentThread();
       setBlocker(t, blocker);
       UNSAFE.park(true, deadline);
       setBlocker(t, null);
   }
   ```

   

唤醒方法是 unpark 方法，接收需要唤醒的线程。

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

从上面的代码也可以看出 LockSupport 类主要调用 UNSAFE 类中方法实现了对线程得阻塞和唤醒。



### 类变量

`AbstractQueuedSynchronizer` 类中成员变量：

```java
private volatile int state;
private transient volatile Node head;
private transient volatile Node tail;
```

`state` 同步状态，为 0 的时表示处于无锁的状态，大于 0  的时候表示已经有线程获取了锁，可以认为当前线程加了 state 次锁。

`head` 等待队列的头结点，表示正在执行的节点。

`tail` 等待队列的为结点。

其中 `head` 和 `tail` 都是 Node 类型的，Node 类型是 AQS 的等待队列中使用的数据类型。主要成员变量如下：

```java
volatile int waitStatus;
volatile Node prev;
volatile Node next;
volatile Thread thread;
Node nextWaiter;
```

`waitStatus` 等待状态，默认值是0, 表示正常的同步节点。1 表示 CANCELLED 由于在同步队列中等待的线程超时或者中断，需要从同步队列中取消等待，节点进入该状态将不会变化。 -1 表示 SIGNAL 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点得以运行。-2 表示 CONDITION 节点在等待中，节点线程等待在 Condition 上，当其他线程对 Condition 调用了 signal() 方后，该节点会从等待队列转移到同步队列中，假如到对同步状态的获取。-3 表示 PROPAGATE 表示下一次共享式同步状态会被无条件地传播下去。

`prev` 当前节点的前驱节点。

`next` 当前节点的后继节点。

`thread` 当前节点存放的线程。

`nextWaiter` 等待队列的后继节点。如果当前节点是共享的，那么这个字段就是一个 SHARED 常量，也就说节点类型（独占或者共享）和等待队列中的后继节点共用同一个字段。



### CLH 锁

CLH 锁是一个基于链表的可扩展、高性能、公平的自旋锁，只能再本地变量上申请线程，通过不断轮询前驱的状态判断是否需要结束自旋。

AQS 同步队列的基本结构如下图所示：

![AQS 同步队列基本结构](https://i.loli.net/2019/08/25/tyZNcxMV8zJPmWC.png)

AQS 将节点加入到同步队列的过程如下图所示：

![AQS 添加节点到同步队列](https://i.loli.net/2019/08/25/gz1vseY9fCRBoiS.png)



AQS 同步队列中首节点是获取同步状态成功的节点，首节点释放同步状态时将会唤醒后继节点，后继节点在获取同步状态成功时将自己设置成为首节点，过程如下图所示：

![AQS 释放同步状态](https://i.loli.net/2019/08/25/hgTnJHQBE6jI1Ri.png)



### 类函数

对于同步状态 `state` 方法的操作有以下三个：

```java
// 获取当前同步状态
protected final int getState(){}
// 设置当前同步状态
protected final void setState(int newState) {}
// 使用 CAS 设置当前状态，该方法能够保证状态设置的原子性
protected final boolean compareAndSetState(int expect, int update) {}
```

AQS 提供的可重写的方法如下：

```java
// 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，
// 然后再进行 CAS设置同步状态。
protected boolean tryAcquire(int arg) {}
// 独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态。
protected boolean tryRelease(int arg) {}

// 共享式获取同步状态，返回大于等于0的值表示获取成功，反之获取失败。
protected int tryAcquireShared(int arg) {}
// 共享式释放同步状态
protected boolean tryReleaseShared(int arg) {}
// 当前 AQS 是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占
protected boolean isHeldExclusively() {}
```


主要的成员方法有以下：

```java
// 获取和设置当前的同步状态
protected final int getState(){}
protected final void setState(int newState){}
// 使用 CAS 设置当前状态，保证原子性
protected final boolean compareAndSetState(int expect, int update){}
// 独占式获取同步状态，获取成功，则其他线程需要等待该县城释放同步状态
protected boolean tryAcquire(int arg) {}
// 独占式释放同步状态
protected boolean tryRelease(int arg) {}
// 共享式获取同步状态，返回值大于 0 则表示获取成功，否二获取失败。
protected int tryAcquireShared(int arg) {}
// 共享式释放同步状态
protected boolean tryReleaseShared(int arg) {}
// 当前同步器是否在独占模式下单线程占用，一般表示是否当前线程所独占
protected boolean isHeldExclusively() {}
// 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则将会进入同步队列等待，该方法将会调用可重写的 tryAcquire 方法
public final void acquire(int arg) {}
// 与 acquire 相同，但是该方法相应中断，当前线程获得同步状态进入到同步列表中，如果当前线程被中断，则该方法抛出 InterruptedException 异常返回
public final void acquireInterruptibly(int arg)
            throws InterruptedException {}
// 超时获取同步状态，如果当前线程在 nano 时间内没有获取到同步状态，那么将会返回 flase, 如果获取返回 true
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {}
// 共享式获取同步状态，响应中断
public final void acquireShared(int arg) {}
// 共享式获取同步状态，响应中断
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {}
// 共享式获取同步状态，增加超时限制
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {}
// 独占式释放同步状态，该方法在释放同步状态后，将同步队列的第一个节点包含的线程唤醒
public final boolean release(int arg) {}
// 共享式释放同步状态
public final boolean releaseShared(int arg) {}

```

AQS 提供的模板方法如下：

```java
/**
独占式获取同步状态，如果当前线程获取同步状态成功，
则由该方法返回，否则进入同步队列等待。
*/
public final void acquire(int arg) {
    // 调用重写的 tryAcquire 方法
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt(); // 中断当前线程
}
// 创建一个 Node 并加入到同步队列中
private Node addWaiter(Node mode) {
    // 新建 Node
    Node node = new Node(Thread.currentThread(), mode);
    // 获取同步队列最后一个元素
    Node pred = tail;
    if (pred != null) {
        // 设置当前元素的前驱元素为最后一个元素
        node.prev = pred;
        // 使用 CAS 设置当前 Node 到末尾
        if (compareAndSetTail(pred, node)) {
            // 设置旧的末尾元素的后继元素为新创建的元素
            pred.next = node;
            return node;
        }
    }
    // 如果以上步骤没有设置成功，则调用该方法加入到队列中
    enq(node);
    return node;
}
// 天界一个元素到队列中，如果队列不存在在初始化队列
private Node enq(final Node node) {
    for (;;) {
        // 获取末尾的 Node
        Node t = tail;
        // 如果末尾的 Node 为空，则需要执行初始化
        if (t == null) { // Must initialize
            // 使用空 Node 初始化 head
            if (compareAndSetHead(new Node()))
                tail = head; // 设置末尾元素等于 head
        } else {
            // 设置新建元素的前驱元素为 原末尾的 Node
            node.prev = t;
            // 设置末尾 Node
            if (compareAndSetTail(t, node)) {
                // 设置原末尾 Node 的后继元素为新建的 Node
                t.next = node;
                return t;
            }
        }
    }
}

// 加入到同步队列中
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取当前元素的前驱元素
            final Node p = node.predecessor();
            // 如果前驱元素为 head 则尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 如果获取到锁则设置 head
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 判断当获取失败的时候是否需要阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 设置 head
private void setHead(Node node) {
    // 设置 head 为当前 Node
    head = node;
    // 清空当前的Node 存储的 Thread
    node.thread = null;
    // 清空前驱元素的连接
    node.prev = null;
}
// 判断获取失败的时候是否需要阻塞
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前驱元素的等待状态
    int ws = pred.waitStatus;
    // 如果等待状态是 SIGNAL，则阻塞
    if (ws == Node.SIGNAL)
        /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
        return true;
    // 如果等待状态大于 0，表明前驱节点是 cancelled，跳过这些元素
    if (ws > 0) {
        /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
        // 使用 CAS 设置 等待状态
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// 取消获取
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

```java
// 与 acquire 方法相同，但是该方法相应中断，当前线程未获取到同步状态而进入
// 同步队列，如果当前线程被中断，则刚发抛出 InterruptedException 异常
public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg); // 如果没有获取到则尝试响应中断式获取
}

// 响应中断式的获取
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    // 尝试初始化 Node 并添加掉同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



```java
// 在 acquireInterruptibly 的基础上增加了超时限制，
// 如果当前线程在超时时间内没有获取到同步状态，
// 那么将会返回 false，如果获取到返回 true
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
     throws InterruptedException {
     if (Thread.interrupted())
         throw new InterruptedException();
     return tryAcquire(arg) ||
         doAcquireNanos(arg, nanosTimeout);
 }
// 获取锁
private boolean doAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 根据超时设置获取截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 获取剩余的时间
            nanosTimeout = deadline - System.nanoTime();
            // 如果剩余的时间小于 0 ，则返回 false 表示未成功获取
            if (nanosTimeout <= 0L)
                return false;
            // 如果应该阻塞且剩余的时间大于设定的超时阈值，则阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



```java
// 共享式获取同步状态，如果当前线程未获取到同步状态，将会进入到同步队列等待，
// 与独占式获取的主要区别是同一时刻可以有多个线程获取到同步状态
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

// 尝试获取共享锁
private void doAcquireShared(int arg) {
    // 创建 Node 并添加到同步队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 获取共享锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 设置 head
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 设置 head
private void setHeadAndPropagate(Node node, int propagate) {
     Node h = head; // Record old head for check below
    // 设置 head
     setHead(node);
     /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
    // 检查后续 Node 是否是共享锁
     if (propagate > 0 || h == null || h.waitStatus < 0 ||
         (h = head) == null || h.waitStatus < 0) {
         Node s = node.next;
         if (s == null || s.isShared())
             doReleaseShared();
     }
 }

// 释放共享锁
private void doReleaseShared() {
    /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
// 唤醒后续节点
private void unparkSuccessor(Node node) {
    /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```



```java
// 获取共享锁响应中断
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



```java
// 在 acquireSharedInterruptibly 的基础上增加了 超时时间
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



```java
// 独占式的释放同步状态，该方法啊会在释放同步状态之后，
// 将同步队列中的第一个节点包含线程唤醒
public final boolean release(int arg) {
    // 调用 tryReleae 释放，
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```



```java
// 共享式释放 
public final boolean releaseShared(int arg) {
     if (tryReleaseShared(arg)) {
         doReleaseShared();
         return true;
     }
     return false;
 }

```



参考：

1. [Java的LockSupport.park()实现分析](https://blog.csdn.net/hengyunabc/article/details/28126139)
2. [不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)
3. [Java并发编程的艺术](https://book.douban.com/subject/26591326/)
4. [Java并发编程实战](https://book.douban.com/subject/10484692/)
5. [【死磕Java并发】—–J.U.C之AQS（一篇就够了）](https://juejin.im/entry/5ae02a7c6fb9a07ac76e7b70)
6. [Java并发之AQS源码分析（一）](http://objcoding.com/2019/05/05/aqs-exclusive-lock/)