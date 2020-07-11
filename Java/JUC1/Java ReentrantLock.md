---
title: "Java ReentrantLock"
tags: ["java", "JUC"]
categories: ["Java"]
date: "2019-01-01T23:00:00+08:00"
---

ReentrantLock 重入锁，即支持重复进入的锁，该锁能够支持一个线程对一个资源反复的加锁。实现可重入锁需要解决两个问题：

1. 线程再次获取锁。锁需要识别获取锁的线程是否是当前占用锁的线程，如果是则成功获取。
2. 锁的最终释放。线程重复多次获取锁，那么需要在相同次数释放该锁后，其他线程能更获取到该锁。加锁时计数器递增，释放锁计数器递减，计数器为 0 的时候表示锁已经成功释放。

ReentrantLock  对于锁有公平锁和非公平锁两种实现，本质上都是基于 AQS，公平锁和非公平锁各自实现了一套获取锁的方法。在 ReentrantLock   中可以通过构造方法的参数指定使用公平锁还是非公平锁。

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

ReentrantLock   使用的同步器继承体系如下：

![同步器继承体系](https://i.loli.net/2019/08/26/QpS1cvh68MreVzT.png)

### 公平锁

ReentrantLock  对于公平锁的实现主要是内部类 FairSync 。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    // 获取锁，调用父类中的 acquire 方法
    final void lock() {
        acquire(1);
    }

    /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
    // 公平锁 - 尝试获取锁，该方法在 acquire 方法中调用
    protected final boolean tryAcquire(int acquires) {
        // 获取当前线程
        final Thread current = Thread.currentThread();
        // 获取同步器状态
        int c = getState();
        // 如果是 0 表示没有锁
        if (c == 0) {
            // 判断是否等待队列否是为空，如果为空则尝试获取锁
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //  如果等待队列为空并且成功的获取了锁，设置当前线程获取到了锁
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果有锁且是当前线程取得的锁则将同步器状态 +1 
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

// 判断是否已经有线程在排队等待获取锁
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}

// 设置独占锁的拥有者
protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}

// 获取独占锁的拥有者
protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
}
```



```java
// 释放锁的方法
protected final boolean tryRelease(int releases) {
    // 根据释放锁的数量，等待释放锁后同步器状态的值
    int c = getState() - releases;
    // 如果当前线程不等于独占锁线程的拥有者，则表明出问题了，抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 如果同步状态值为 0 则表明该线程用完锁了，则设置线程拥有者为 null
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 设置状态
    setState(c);
    return free;
}
```



### 非公平锁

ReentrantLock  对于公平锁的实现主要是内部类 NonfairSync。

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
    // 加锁
    final void lock() {
        // 首先尝试获取锁，获取到了则设置当前线程为独占锁的拥有者
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else`// 未获取到锁，则继续走获取锁的流程
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

// 非公平方式获取锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 这里是和公平锁区别的一个地方，如果发现没有锁，直接尝试获取，
        // 不管是否同步队列中是否线程等待锁。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



### 与 synchronized  的比较

ReentrantLock   在加锁和内存上提供的语义与 synchronized  相同，不过此外其还提供了包括定时等待，可中断锁的等待，公平性，已经实现非块结构的加锁，性能上也是优于内置锁的。

但是使用 ReentrantLock 的危险性更高，应该在内置锁满足不了需求的时候才考虑使用 ReentrantLock 。另外 synchronized  是 JVM 的内置属性，可以执行一些优化例如对线程封闭对象的锁消除，通过增加锁的粒度来消除内置锁等等。



参考：

1. [Java并发编程的艺术](https://book.douban.com/subject/26591326/)

