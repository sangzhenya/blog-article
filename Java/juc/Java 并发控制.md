---
title: "Java 并发控制"
tags: ["java", "JUC"]
categories: ["Java"]
date: "2019-01-01T22:00:00+08:00"
# toc: true
---

### 锁

#### 公平锁和非公平锁

并发包中 ReentrantLock 的创建可以指定构造函数的 boolean 类型来得到公平锁赫尔非公平锁，默认是非公平锁。

##### 二者区别

**公平锁：**Thread acquire a fair lock in the order in which they request it.

公平锁：在并发环境中，每个线程在获取锁时先查看此锁维护的等待队列，如果为空，或者当前线程是等待队列的第一个就占用锁，否则就加入到队列中，按照 FIFO 的规则从队列中取出自己。

**非公平锁：** A nonfair lock permit barging: threads requesting a lock can jump ahead of the queue of waiting threads if the lock happens to be available when it is requested.

非公平锁直接尝试占有锁，如果尝试失败再使用类似公平锁的方式。

**备注：**非公平锁的有点在于吞吐量比公平锁大，synchronized 也是一种非公平锁。

#### 可重入锁

指的是在同一线程外层函数获得锁之后，内存递归函数仍然能获取该锁的代码，在同一个线程在外层方法获取锁的时候，在进入内存方法会自动获取锁。即线程可以进入任何一个已经拥有的锁所同步着的代码块。

#### 自旋锁

尝试获取锁的线程不会立即阻塞，而是采用循环的方式尝试获取锁，这样的好处是减少上下文切换的消耗，缺点是会消耗 CPU。

#### 独占锁和共享锁

**独占锁**：指该锁一次只能被一个线程所持有。对 ReentrantLock 和 Synchronized 而言都是独占锁

**共享锁：**指该锁可以被多个线程所持有，对 ReentrantReadWriteLock 其读锁时共享锁，写锁时独占锁。读锁的共享锁可以保证并发读是非常高效的，读写和写读，写写的过程是互斥的。



### 三种 JUC 提供的并发控制类

#### CountDownLatch 

让一些线程阻塞直到另一些线程完成一系列操作后才被唤醒。 CountDownLatch 主要有两个方法，当一个或多个方法调用 await 方法时，调用线程会被阻塞。其他线程调用 countDown 方法会将计数器减 1， 调用 countDown 方法的线程并不会阻塞，当计数器的值变为0 的时候，因调用 await 方法的线程会被唤醒，秩序执行。

#### CyclicBarrier

让一些线程阻塞到另外一些线程都完成后才被唤醒。CyclicBarrier 主要有一个方法，当一个或多个方法调用 await 方式十，调用线程会被阻塞。当前调用 await的线程数量达到一定数量的时候，唤醒某一个线程开始执行。

#### Semaphore

信号量主要用于两个目的，一个是用于多个共享资源的互斥使用，另一个是用于并发线程数的控制。



### 阻塞队列

在多线程领域，阻塞即在某些情况下回挂起线程（阻塞），一旦条件满足，被挂起的线程又会被自动唤醒。

有了阻塞队列我们就不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程，一切都可以交给 BlockingQueue 处理。

1. ArrayBlockingQueue
2. LinkedBlockingQueue
3. PriorityBlockingQueue
4. SynchronousQueue



### synchronized 和 Lock 区别

#### 原始构成不同

synchronized 是关键字属于 JVM 层面的，使用 monitorenter 和 monitorexit 完成，底层是monitor 对象完成其实 wait/notify 等方法也是依赖于 monitor 对象，只有在同步代码块或方法中才能调用 wait/notify 等方法。

Lock 是具体的类是 api 层面的。

#### 使用方法不同

synchronized 不需要用户手动释放锁，当 synchronized 代码执行完成后会自动让线程释放对锁的占用。

Lock 则需要用户手动释放锁，若没有释放锁，就可能导致出现死锁现象，需要 lock 和 unlock 方法配合 try/finally 语句块使用。

#### 等到是否可中断 

synchronized 不可中断，除非抛出异常或者运行完成。

Lock 可中断，设置超时方法 trylock(long timeout, TimeUnit unit)， lockInterruptibly 放代码块中，调用 interrupt 方法中断。

#### 加锁是否公平

synchronized 非公平锁。

Lock 可以是公平锁也可以使用非公平锁。

#### 锁绑定多个条件 Condition

synchronized 没有

Lock 用来实现分组唤醒需要唤醒的线程，可以精确唤醒，而不是像 synchronized 一样要吗随机唤醒，要么唤醒全部。



### 线程池

线程池做的工作主要是控制运行线程的数来，处理过程将任务放入队列，然后在线程创建后启动这些任务，如果线程数量超过了最大数量超出数量的线程排队等候，等其他线程执行完毕，再从队列中取出任务来执行。

主要特点为：线程复用，控制最大并发数量，管理线程。

1. 降低资源消耗。通过重复利用已经创建的线程降低线程创建和消耗造成的消耗。
2. 提供响应速度。当任务到达的时候，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程时长稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配，调优和监控。

#### 线程池类结构

![线程池类结构](https://i.loli.net/2019/06/12/5d00e2ef0288389976.png)

#### 使用线程池新建线程

```java
ExecutorService threadPool = Executors.newFixedThreadPool(3);
ExecutorService threadPool = Executors.newSingleThreadExecutor();
ExecutorService threadPool = Executors.newCachedThreadPool();
ExecutorService threadPool = new ThreadPoolExecutor(3, 5, 300, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
```

#### 线程池7个参数的意义

##### corePoolSize

创建了线程池之后，当有请求任务来之后，就会安排池中的线程去执行请求任务，近似理解为今日当值线程，当线程池中的线程数目达到 corePoolSize 后，就会把到达的任务放到缓存队列中。

##### maximumPoolSize

线程池能够容纳同时执行的最大线程数，此值必须大于等于1。

##### keepAliveTime

多余的空闲线程的存活时间。当前线程池数量超过 corePoolSize 时，当空闲时间达到 keepAliveTime 值时，多余空闲线程会被销毁只剩下 corePoolSize 个线程为止。

##### unit

keepAliveTime 的单位

##### workQueue

任务队列，被提交但尚未被执行的任务。

##### threadFactory

表示生成线程池中工作线程的线程工厂，用于创建线程一般使用默认即可。

##### handler

拒绝策略，表示当队列满了并且线程大于等于线程池最大线程数的拒绝策略。



### 线程池的底层原理

#### 处理流程

![线程池处理流程](https://i.loli.net/2019/06/12/5d00f453ada5429341.png)



1. 在创建线程池后，等待提交过来的任务请求。
2. 当调用 execute() 方法添加一个请求任务时，线程池会做如下判断：
   1. 如果正在运行的线程数量小于 corePoolSize 那么马上创建新的运行这个任务。
   2. 如果正在运行的线程数量大于或等于 corePoolSize 那么将这个任务放入队列。
   3. 如果这个时候队列满了掐正在运行的线程数量还小于 maximumPoolSize 那么还是要创建非核心线程立即运行这个任务。
   4. 如果队列满了且正在运行的线程数量大于或等于 maximumPoolSize 那么线程池就会启动饱和拒绝策略来执行。
3. 当一个线程完成任务时，它会从队列中取出下一个任务来执行。
4. 但一个线程无事可做超过 keepAliveTime 时，线程池会判断如果当前运行线程数大于 corePoolSize 那么这个线程会被停掉。所以线程池所有任务完成后它最终会收缩到 corePoolSize 的大小。

#### 最大线程数量参数设置

##### CPU  密集型

CPU 密集型即任务需要大量的运算，而没有阻塞，CPU 一直全速运行。CPU 密集任务只有在多核 CPU 上才能得到加速。一般设置： CPU 核心数 + 1 个线程的线程池。

##### IO 密集型

IO 密集型即任务需要大量的 IO，即大量的阻塞。在单线程运行 IO 密集型的任务会导致浪费大量的 CPU 运算能力在等待。所以在 IO 密集型任务中使用多线程可以大大加速程序运行，即使在 单核心 CPU 上，这种加速的主要就是利用了被浪费掉的阻塞时间。

1. CPU 核心数 * 2
2. CPU 核心数  / （ 1 -  阻塞系数 ）   阻塞系数在 0.8 - 0.9 之间



### 拒绝策略

#### AbortPolicy

直接抛出 RejectedExecutionException 异常阻止系统正常运行。

#### CallerRunsPolicy

调用者运行 一种调节机制，该策略既不会抛弃任务，也不会抛出异常，而是将某些任务回退到调者，从而降低调用流量。

#### DiscardOldestPolicy

抛弃任务队列中等待最久的任务，然后当前任务加入到队列中尝试再次提交当前任务。

#### DiscardPolicy

直接丢弃任务，不予任何处理也不抛出异常。如果允许任务丢失，着市场最好的一种方案。



### 死锁

死锁是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力干涉那他们将无法推进下去，如果系统资源充足，进程资源请求都能够得到满足，死锁出现的可能性很低，否则就会因争夺有限资源而陷入死锁。

**jps.exe 查看进程**

![jps](https://i.loli.net/2019/06/12/5d010011a49ca54076.png)

**jstack.exe 查看栈使用情况**

![jstack](https://i.loli.net/2019/06/12/5d01007eb775878456.png)



