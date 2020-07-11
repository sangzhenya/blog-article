---
title: "Java GC 2"
tags: ["Java", "JVM"]
categories: ["Java"]
date: "2020-02-25T09:00:00+08:00"
---

几种垃圾回收器：

![几种垃圾回收器对比](https://i.loli.net/2019/06/19/5d0a2546f00be94890.png)

### Serial

**串行垃圾回收器**

为单线程环境设计且只可以使用一个线程进行垃圾回收，会暂停所有用户线程，直到收集结束。所以不适合服务器环境。

![串行收集器](https://i.loli.net/2019/06/19/5d0a2f4ab1a1d51946.png)

**Serial** 串行收集器是最古老，最稳定已经效率最高的收集器，只使用一个线程去回收，但是其在垃圾回收过程中可能产生较长的停顿（Stop-The-Word 状态）。虽然在垃圾收集过程中需要暂停其他工作线程，但是它简单高效，对于限定单核 CPU 环境来说，没有线程交互的开销可以获得最高的单线程垃圾收集器，因此 Serial 垃圾收集器依然是 Java 虚拟机运行在 Client 模式下默认的新生代垃圾收集器。

对应的 JVM 参数是： `-XX:UseSerialGC` 开启之后会使用 Serial (Young 区使用) + Serial Old (Old 区使用) 的收集器组合。即新生代和老年代都会使用串行收集器，新生代使用复制算法，老年代使用标记整理算法。

**Serial Old** 是 Serial 垃圾收集器的老年代版本，它同样是一个单线程的收集器，使用 标记整理算法，这个收集器也是主要运行在 Client 默认的 Java 虚拟机默认的老年代垃圾收集器。

在 Server 模式下，主要有两个用处：

1. 在 JDK 1.5 之前与 新生代 Paralegal Scavenge 收集器搭配使用
2. 作为老年代版中使用 CMS 收集器的后背垃圾收集方案。



### Parallel

**并行垃圾收集器**

多个垃圾收集器线程并行工作，此时用户线程是暂停的（Stop-The_World 状态）。适用于科学计算/大数据处理后台处理等弱交互场景。

![并行垃圾收集器](https://i.loli.net/2019/06/19/5d0a30c27422c18523.png)

**ParNew** 收集器其实就是 Serial 收集器新生代的并行多线程版本，最常用的是配合老年代的 CMS GC 工作，其余行为和 Serial 收集器完全一样， ParNew 垃圾收集器在垃圾收集过程中同样也要暂停所有其他工作线程。是很多 Java 虚拟机运行在 Server 模式下新生代的默认垃圾收集器。

对应的 JVM 参数： `-XX:+UseParNewGC`  启用 ParNew 收集器，只影响新生代收集不影响老年代。开启上述参数后，会使用：ParNew (Young 使用) +  Serial Old （Old 区使用） 的收集器组合。即新生代使用复制算法，老年代使用标记正常算法。

但是 ParNew + Tenured 这样的搭配，java 8 已经不再被推荐 使用会用 Warning：

`Using the ParNew young collector with the Serial old collector is deprecated and will likely be removed in a future release`

另外可以使用 `-XX:ParallelGCThreads` 限制线程数量，默认开启和 CPU 数目相同数量的线程数。

![并行收集器](https://i.loli.net/2019/06/19/5d0a32f3373ba39772.png)

**Parallel Scavenge** 收集器类似 ParNew 也是一个新生代的垃圾收集器，使用复制算法，也是一个并行的多线程垃圾收集器，又称吞吐量优先收集器。即串行收集器在新生代和老年代的并行化。有以下两个重点：

1. 可控吞吐量。高吞吐量意味着高效利用 CPU 的时间，多用于后台运算而不需要太多交互的任务。（吞吐量：运行用户代码时间 / （运行代码时间 + 垃圾收集时间））
2. 自适应调节策略。这是一个 ParalleScavenge 收集器与 ParNew 收集器的一个重要区别。虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或最大的吞吐量（`-XX:MaxGCPauseMillis`）。

对应的 JVM 参数：`-XX:+UseParalleGC` 或 `-XX:UseParallelOldGC` 可以相互激活。会使用 ParalleGC (新生代使用) + ParalleOldGC （老年代使用） 的组合。即新生代使用复制算法，老年代使用标记整理算法。

此外： `-XX:ParallelGCThreads=N` 表示启动多少个 GC 线程。默认 CPU > 8 则 N = 5/8，否则等于实际个数。

**Parallel Old ** 收集器是 Parallel Scavenge 的老年代版本，使用的是多线程的标记整理算法，Parallel 收集器是在JDK 1.6 才开始提供的。在 JDK 1.6 之前，新生代使用 Parallel Scavenge 收集器只能搭配老年代的 Serial Old 收集器，只能保证新生代的吞吐量优先，无法保证整体吞吐量优先。

Parallel Old 正是为了在老年代同样提供吞吐量优先的垃圾收集器，如果系统对吞吐量要求比较高， JDK 1.8 后可以优先考虑新生代使用 Parallel Scavenge 和老年代 Parallel Old 收集器的搭配策略。

对应 JVM 参数：`-XX:+UseParallelOldGC` 使用 Parallel Old 收集器，设置该参数后，新生代使用 Parallel + 老年代 Parallel Old。



### CMS

**并发垃圾收集器**

用户线程和垃圾收集线程同时执行（不一定是并行，可能是交替执行），不需要停顿用户线程，互联网公司多使用此，适应于对响应时间有要求的场景。

![CMS 垃圾收集器](https://i.loli.net/2019/06/19/5d0a3899d474e57297.png)

Concurrent Mark Sweep 标记并发清楚，是一种以获取最短回收停顿时间为目标的收集器。适合应用在互联网在互联网或者 B/S 系统的服务器上，这类应用尤其重视服务器的响应速度，希望系统停顿时间最短，CMS 非常适合堆内存大，CPU 核心数多的服务段应用，也是 G1 出现之前大型应用的首选收集器。Concurrent Mark Sweep 并发标记清除，并发收集低停顿，并发是指与用户线程一起执行。

对应 JVM 参数： `-XX:UseConcMarkSweepGC` 开启该参数后会自动将 `-XX:+UseParNewGC` 打开，表示使用 ParNew (新生代使用) + CMS (老年代使用) + Serial Old 的收集器组合，Serial Old 将作为 CMS 出错的后备处理器。

清理步骤分为以下四步：

1. 初始标记（Initial Mark）

   只是标记以下 GC Roots 能直接关联的对象，速度很快，仍然需要暂停所有的工作线程。

2. 并发标记（Concurrent Mark） 和用户线程一起。

   进行 GC Roots 跟踪过程，和用户线程一起工作，不需要暂停工作线程。主要用于标记过程，标记全部对象。

3. 重新标记（Remark）

   为了修正在并发标记期间，因用户程序继续运行而导致标记产生的变动的那一部分对象的标记记录，仍然需要暂停所有工作线程。由于并发标记时用户线程依然在运行，因此在正式清理前再修正。

4. 并发清除 （Concurrent Sweep）

   清楚 GC Roots 不可达对象，和用户线程一起工作，不需要暂停工作线程。基于标记结果，直接清理对象。

由于耗时最长的并发标记和并发清除过程中，垃圾回收线程可以和用户线程一起并发工作，所以总体上来看 CMS 收集器的内存回收和用户线程是一起并发地执行的。



![四步简介](https://i.loli.net/2019/06/19/5d0a3b84834c823605.png)

优点是：并发收集低停顿。

缺点是：并发执行，对 CPU 资源压力大。由于并发进行，CMS 在收集与清理线程同时会增加对堆内存的占用。即 CMS 必须要在 老年代堆用尽之前完成垃圾回收，否则 CMS 回收失败时，将触发担保机制，串行老年收集器会以 STW 的方式进行一次 GC， 从而造成交大昂的停顿时间。

采用的标记清除会算法会导致大量碎片。标记清除算法无法整理空间碎片，老年代空间会随着应用时长被逐步耗尽，最后不得不通过担保机制对堆内存进行压缩。CMS 也提供了 `-XX:CMSFullGCsBeForeCompaction` 参数来指定多少此 CMS 收集之后进行一次压缩的 FullGC。默认值是0 ，即每次都进行内存整理。

### G1

将堆内存分割成不同的区域，然后并发的对其进行垃圾回收。

之前垃圾回收器的特点：

1. 年轻代和老年代是各自独立且连续的内存块
2. 年轻代收集使用 单 eden + s0 + s1 进行复制算法
3. 老年代收集必须扫描整个老年代区域
4. 都是以尽可能的少而快速地执行 GC 为设计原则。

而 G1 （Garbage First） 收集器 是一款面向服务端的收集器。主要应用在多处理器和大内存环境中，在实现高吞吐量的同时，尽可能的满足垃圾收集时间的要求。其有以下特点： 

1. 像 CMS 收集器一样，能与应用程序并发执行
2. 整理空闲空间更快
3. 需要使用更多的时间预测 GC 停顿时间
4. 不希望牺牲大量的吞吐性能
5. 不希望更大的 Java Heap

G1 收集器设计的目的就是取代 CMS 收集器，它与 CMS 相比，在以下方面表现的更出色：

1. **其是一个具有整理内存过程的垃圾收集器，不会产生很多的内存碎片。**
2. **Stop The World 更可控，G1 在停顿时间上添加了预测机制，用户可以指定期望停顿时间。**

CMS 垃圾收集器虽然减少了暂停应用程序运行的时间，但是它还是存在着垃圾碎片问题。于是为了去除内存碎片问题，同时有保留 CMS 垃圾收集器降低暂停时间的优点，Java 7 发布了新一代的垃圾收集器 G1 垃圾收集器。

主要改变是 Eden Survivor 和 Tenured 等内存区域不再是连续的了，而是变成了一个个大小一样的 Religion，每个 region 从 1m 到 32m 不等。一个 region 有可能属于 Eden， Survior 或者 Tenured 内存区域。

特点：

1. **重分利用多CUP，多核环境的硬件优势，尽量缩短 STW.**
2. **整理顺颂标记整理算法，局部使用复制算法，不会产生内存碎片**
3. **宏观上不在区分年轻代和老年代，把内存划分成多个独立的子区域，近似理解为一个围棋棋盘。**
4. **整个内存区都混合在一起。但其本身依然存在小范围内进行年轻代和老年代区分，保留了新生代和老年代，但他们不再是物理隔离，而是一个部分 Region 的集合，且不需要 Region 是连续的，也就是依然会采用不同的方式处理不同的内存区域。**
5. **其虽然是分代收集器，但是整个内存不存在物理上的年轻代和老年代的区别，也不需要完全独立的 Survior 堆来做复制准备。只有逻辑上的分代概念，或者说每个分区都可能随着 G1 的运行在不同代之间前后切换。**

底层原理：

Region 区域化的垃圾收集器，最大的好处是化整为零，避免全内存扫描，只需要按照区域来进行扫描即可。区域化内存划片 Region，整体编为了一系列不连续内存区域，避免了全内存区GC 操作。核心思想是将整个内存区域化成大小不同的子区域，在 JVM 启动时会自动设置这些子区域的大小。在堆使用上，G1 并不要求对象的存储一定是物理上连续的只要逻辑上连续即可，每个分区也不会固定的为某个代服务，可以按需在年轻代和老年代之间切换。启动时可以通过参数 `-XX:G1HeapRegionSize=n` 可以指定分区大小 （1m ~ 32m，且必须是2的幂），默认将整个堆最多划分为 2018 个分区。大小范围在 1MB ~ 32MB，最多能设置 2018 个区域，也即能够支持最大的内存为：32MB*2048 = 65526MB = 64G 内存。

![G1 底层原理图](https://i.loli.net/2019/06/24/5d10c13aae32521154.png)

G1 算法将堆划分为若干个区域(Region)，它仍然属于分代收集器。这些 Region 的一部分包含新生代，新生代的垃圾收集器仍然采用暂停所有应用线程的方式，将存活对象复制到老年代或者 Survior 空间。这些 Region 的区域一部分包含老年代，G1 收集器通过将对象从一个区域复制到另外一个区域完成清理工作，也就意味着，在正常的处理过程中，G1 完成了堆的压缩，也就不会有 CMS 内存碎片的问题的存在了。

G1 中还有一种特殊区域，叫做 Humongous (巨大的)区域，如果一个对象占用空间超过分区容量的 50% 以上，G1 收集器就认为这是一个巨大的的对象。这些巨型对象默认会直接分配在老年代，但是如果它是一个短期存在的巨型对象，就会对垃圾收集造成负面影响。为了解决这个问题，G1 划分一个 Humongous 区，它用量专门存放巨型对象。如果一个 H 区域放不下巨型对象，那么 G1 会寻找连续的 H 分区来存储。为了能给找到连续 H 区，有时候不得不启动 Full GC。



参考：[深入理解 Java G1 垃圾收集器](http://ghoulich.xninja.org/2018/01/27/understanding-g1-garbage-collector-in-java/)

**对象分配策略**

说起大对象的分配，我们不得不谈谈对象的分配策略。它分为 3 个阶段：

1. TLAB（Thread Local Allocation Buffer）线程本地分配缓冲区

2. Eden 区中分配

3. Humongous 区分配

TLAB 为线程本地分配缓冲区，它的目的是为了使对象尽可能快的分配出来。如果对象在一个共享的空间中分配，我们需要采用一些同步机制来管理这些空间内的空闲空间指针。在 Eden 空间中，每一个线程都有一个固定的分区用于分配对象，即一个 TLAB。分配对象时，线程之间不再需要进行任何的同步。

对 TLAB 空间中无法分配的对象，JVM 会尝试在 Eden 空间中进行分配。如果 Eden 空间无法容纳该对象，就只能在老年代中进行分配空间。

最后，G1 提供了两种 GC 模式，Young GC 和 Mixed GC，两种都是 Stop The World（STW）的。下面我们将分别介绍一下这 2 种模式。

**Young 区回收**

针对于 Eden 区进行收集，Eden 区耗尽后会被触发，主要是小区域收集 + 形成连续的内存块，避免内存碎片。Eden 区域数据移动到 Survior 区，假如 Survior 去空间不够，Eden 区数据会部分晋升到 Old  区。Survior 区的数据移动到新的 Survivor 区，部分数据晋升到 Old 区。最后 Eden 区收拾干净了，GC 结束，用户的应用程序继续执行。步骤如下：

1. 根扫描：扫描静态和本地对象
2. 更新 RS： 处理  Dirty Card 队列和更新 RS
3. 处理 RS：检测从年轻代指向老年代的对象
4. 对象拷贝：拷贝存活的对象到 survivor/old 区域
5. 处理引用队列：处理软引用，弱引用，虚引用

收集前：

![G1 垃圾收集 Eden 区](https://i.loli.net/2019/06/24/5d10c3798ef0132782.png)

收集后：

![G1 垃圾收集器 收集 Eden 区域](https://i.loli.net/2019/06/24/5d10c5766aab387009.png)

**Old 区回收**

四步收集过程：

1. 初始标记：只标记处 GC Roots 能直接关联的对象
2. 并发标记：进行 GC Root Tracing 的过程
3. 最终标记：修正并发标记期间，因程序运行导致标记发生变化的一部分对象。
4. 筛选回收：根据时间来进行价值最大化的回收。

![四个步骤](https://i.loli.net/2019/06/24/5d10c7135be0c56895.png)

```shell
# 常用配置参数
-XX:UseG1GC	# 使用 G1 垃圾收集器
-XX:G1HeapRegionSize=n	# 设置 G1 区域的大小。值是2的幂，范围是 1MB 到 32MB。目标是根据最小的 Java 堆大小划分出月 2048个区域。
-XX:MaxGCPauseMills=n	# 最大 GC 停顿时间，这是一个软目标，JVM 将尽可能（但不保证）停顿小于这个时间。
-XX:InitiatingHeapOccupancyPercent=n	# 堆占用多少时候就触发 GC，默认是45
-XX:ConcGCThread=n	# 并发 GC 使用的线程数
-XX:G1ReservePercentn=n	# 设置作为空闲空间的预留内存百分比，以降低目标空间溢出的风险，默认值是 10%
```





**简单的比较**

![一点比较](https://i.loli.net/2019/06/19/5d0a28d3bb56165022.png)



### 如何选择

+ 单 CPU 或 小内存，单机程序。

  `-XX:UseSerialGC`

+ 多 CPU，需要最大吞吐量，如果后台计算型应用。

  `-XX:+UseParallelGC`

  `-XX:+UseParallelOldGC`

+ 多 CPU，追求低停顿时间，需快速响应如互联网应用。

  `-XX:+UseCOncMarkSweepGC`

  `-XX:+ParNewGC`

| 参数                                          | 新生代垃圾收集器    | 新生代算法             | 老年代垃圾收集器                     | 老年代算法 |
| --------------------------------------------- | ------------------- | ---------------------- | ------------------------------------ | ---------- |
| `-XX:+UseSerivalGC`                           | SerialGC            | 复制                   | SerialOldGC                          | 标记整理   |
| `-XX:+UseParNewGC`                            | ParNew              | 复制                   | SerialOldGC                          | 标记整理   |
| `-XX:UseParallelGC` 或 `-XX:UseParallelOldGC` | Parallel [Scavenge] | 复制                   | ParallelOld                          | 标记整理   |
| `-XX:UseConcMarkSweepGC`                      | ParNew              | 复制                   | CMS + SerialOld                      | 标记清除   |
| `-XX:UseG1GC`                                 | G1                  | 整体上使用标记整理算法 | 局部是通过复制算法，不会产生内存碎片 |            |



### 其他

**查看当前使用的垃圾回收器**

`java -XX:+PrintCommandLineFlags -version`

![查看当前使用的垃圾回收器](https://i.loli.net/2019/06/19/5d0a2b8f4de5960327.png)



**Java GC 回收器的类型**

1. UseSerialGC
2. UseParallelGC
3. UseConcMarkSweepGC
4. UseParNewGC
5. UseParallelOldGC
6. UseG1GC



**不同垃圾回收器使用范围**

![不同垃圾回收器使用范围](https://i.loli.net/2019/06/19/5d0a2d1fe4f1069126.png)

---

![垃圾收集器范围](https://i.loli.net/2019/06/19/5d0a2dff4aafe40898.png)



1. Serial -> Serial Old
2. ParNew -> Serial Old
3. Parallel Scavenge -> Parallel Old
4. CMS -> Parallel Scavenge



---

Server/Client 模式

1. 32 位操作系统，不论硬件如何都默认使用 Client 的 JVM 模式
2. 32 位其他系统，2g 内存同时2个CPU以上会使用 Server 模式，低于该配置还是 Client 模式
3. 64 位操作系统只有 Server 模式。



部分词汇：

+ DefNew: Default New Generation

+ Tenured: Old

+ ParNew: Parallel New Generation

+ PSYoungGen: Parallel Scavenge

+ ParOldGen: Parallel Old Generation