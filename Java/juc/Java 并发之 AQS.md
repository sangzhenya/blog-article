## Java 并发之 AQS

[TOC]

众所周知 synchronized 是重量级锁，虽然在 JDK 1.6 后使用了轻量级锁，偏向锁，自适应自旋锁，锁粗化，锁消除等手段去去优化，但是其基于 JVM 的隐式获取和释放锁的方式导致缺少了获取锁和释放锁的可操作性。如果需要高效的实现锁并操作获取和释放锁的过程则需要使用显式锁 Lock，而其中 JUC 包下的 AQS 类是很多锁组件的基础。

AQS 即 `AbstractQueuedSynchronizer`, 是一个构建锁和同步队列的框架，底层是使用 CAS 保证操作的原子性，利用 FIFO 队列实现线程之间的锁竞争。 AQS 是 `ReentrantLock` ， `CountDownLatch`  和 `Semaphore` 等锁的底层实现机制，其将基础同步相关的抽象细节封装，屏蔽了大量的细节，大大降低了实现同步工具的工作。

### LockSupport

AQS 内部线程调度包含线程的阻塞和唤醒都使用了 LockSupport 这个类，所以首先看一下这个类中的方法。

```java
// 阻塞当前线程，直至调用 unpark 方法或者当前线程被中断
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
public static void park() {
    UNSAFE.park(false, 0L);
}
```

```java
// 阻塞当前线程，直到 nanos 纳秒，或者调用 unpark 方法，或者当前线程被中断
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

```java
// 阻塞当前线程，直到一个绝对的时间，或者调用 unpark 方法，或者当前线程被中断
public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline);
    setBlocker(t, null);
}
```

```java
// 唤醒线程

public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```





### AQS 主要成员变量

```java
private transient volatile Node head;
private transient volatile Node tail;
private volatile int state;
```

header 为等待队列的头结点，tail 为等待队列的尾结点，state 则表示同步状态。大于 0 表示有锁状态，每次加锁都会在原有 state 状态上加 1， 所以可以认为当前线程加了锁的次数。释放锁时 state 减 1，当为 0 的时候表示是无锁状态。成员变量均被 volatile 修饰保证线程之间的可见性。

### AQS 主要成员方法

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



### AQS 队列

AQS 依赖一个 FIFO 双向队列依赖来完成同步状态的管理，如果当前线程获取同步状态失败时，AQS 则会将当前线程已经等待的状态信息构造成一个节点并加入到同步队列中，同时会阻塞当前线程时。当同步状态释放的时候，会把首节点唤醒，使其再次尝试获取同步状态。队列中的每个线程都保存着线程的引用，状态信息，前驱节点，后继节点。



### AQS 独占锁



### AQS 共享锁





参考：

1. [Java并发编程实战](https://book.douban.com/subject/10484692/)
2. [【死磕Java并发】—–J.U.C之AQS（一篇就够了）](https://juejin.im/entry/5ae02a7c6fb9a07ac76e7b70)
3. [Java并发之AQS源码分析（一）](http://objcoding.com/2019/05/05/aqs-exclusive-lock/)