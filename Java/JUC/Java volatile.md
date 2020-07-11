---
title: "Java volatile"
tags: ["java", "JUC", "JVM"]
categories: ["Java"]
date: "2019-01-02T08:00:00+08:00"
---

### JMM

> JVM: Java 虚拟机
>
> JMM: Java 内存模型

Java 内存模型 Java Memory Model 本身是**一种抽象的概念并不真实存在**，它描述的是一组规则或规范，通过这组规范定义了程序中的各个变量（包括实例字段、静态字段和构成数组对象元素）的访问方式。



**JMM 关于同步的规定：**

1. 线程解锁前，必须把共享变量的值刷回主内存
2. 线程加锁前，必须读取主内存到自己的工作内存
3. 加锁和解锁是同一把锁



由于 JVM 运行程序的实体是线程，而每个线程创建时 JVM 都会为其创建一个工作内存（也可称为栈空间），工作内存时每个线程的私有数据区，而 Java 内存卡模型规定所有变量都存储在**主内存**，主内存是共享内存区域，所有线程都可以访问，==但线程对变量的操作（读取赋值等）必须在工作内存中进行，首先要将变量从主内存复制到自己工作内存空间。然后对变量进行操作，操作完成再将变量写会主内存。== 线程不能直接操作主内存中的变量，各个线程的工作内存中存储着主内存的变量**副本拷贝**，不同个线程无法访问对方的工作内存，线程间的通讯必须通过主内存完成，其简要访问过程如下图：

![JMM 缓存控制](https://i.loli.net/2019/06/07/5cf9b171ba79184058.png)

*主内存：物理内存空间；工作内存：CPU 缓存*



### 内存模型三大特性

#### 原子性

​	Java 内存模型保证了 read, load, use, assign, store, write, lock 和 unlock 操作具有原子性，例如对一个 int 类型变量执行 assign 操作，这个操作就是原子性的。但是 Java 内存模型允许虚拟机将没有被 volatile 修饰的 64 位数据 (long double) 的读写操作划分为两次 32 为操作来进行，即 load, store, read 和 write 操作可以不具备原子性。

```java
public class VolatileDemo2 {
    public static void main(String[] args) {
        MyData1 myData = new MyData1();
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    myData.plusOne();
                }
            }, String.valueOf(i)).start();
        }

        while (Thread.activeCount() > 2) {
            Thread.yield();
        }
        // 如果是原子性的话，则最终的计算结果应该是 20000
        System.out.println(Thread.currentThread().getName() + " value ==> " +myData.number);
    }
}

class MyData1 {
    volatile int number = 0;
    void plusOne() {
        number += 1;
    }
}
```

##### 为什么 volatile 不支持原子性？  

number++ 的该操作不支持原子性。源码如下：

```java
void plusOne();
    Code:
       0: aload_0
       1: dup
       2: getfield      #2                  // Field number:I
       5: iconst_1
       6: iadd
       7: putfield      #2                  // Field number:I
      10: return
```

在刷回主内存前（或因线程挂起），该共享值可能已经被修改了。

如果要实现线程安全，则要满足以下两个条件：

1. 对变量的写操作不依赖当前值。
2. 该变量没有包含在具有其他变量的不变式中。

*测试发现在单核 CPU 下 仅仅使用 volatile 也能保证程序输出正确的结果。*



#### 可见性

 一个线程修改了共享变量的值，其他线程能够立即得知这个修改。Java 内存模型是通过在变量修改后将新值同步会主内存，在变量读取前从主内存刷新变量值来实现可见性的。主要有以下三种方式实现可见性：

1. volatile
2. synchronized, 对一个变量执行 unlock 操作前，必须把变量值同步回主内存。
3. final, 被 final 关键字修饰的字段在构造器中一旦初始化完成，并且没有发生 this 逃逸 （其他线程通过 this 引用访问到初始化到一半的对象 ），那么其他线程就能看到 final 对象的值。

```java
public class VolatileDemo {
    public static void main(String[] args) {
        MyData myData = new MyData();
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " ==> start");

            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            myData.addTo60();
            System.out.println(Thread.currentThread().getName() + "==> update value to " + myData.number);
        }, "Test01").start();

        while (myData.number == 0) {

        }

        System.out.println(Thread.currentThread().getName() + " end!");
    }
}

class MyData {
    volatile int number = 0;    // myData number 修改后自动
//    int number = 0;       // 无法结束
    public void addTo60(){
        this.number = 60;
    }
}
```

##### volatile 可见性的原理

1. 修改 volatile 变量时会强制将修改后的值刷新到主内存中
2. 修改 volatile 变量后会导致其他线程工作内存中对应的变量值失效，因此再读取该变量值时需要重新从主内存中读取。

底层实现如下：

Java 代码如下：

```java
instance = new Sinlgeton();
```

转换成 汇编代码如下：

```shell
0x01aedeld: movb $0x0, 0x1104800(%esi);0x01a3de24: lock add1 $0x0, %(esp);
```

有 volatile 变量修饰的共享变量进行写操作的时候回多出 Lock 前缀部分的代码，该指令在多核处理器引发了两件事情：

1. 将当前处理器缓存行的数据写回到系统内存。

   在写操作时，如果访问内存区域已经缓存在处理器内部，则不会声言 LOCK# 信号。相反，其会锁定这块内存区域的缓存并写会到内存，并使用缓存一致性机制确保修改的原子性，此操作被称为 “缓存锁定”， 缓存一致性机制阻止同时修改由两个以上处理器缓存的内存区域数据。

2. 这个写回的操作会使其他 CPU 里缓存了该地址的数据无效。

   处理器使用 MESI 控制协议去维护内部缓存和其他处理器缓存的一致性。在多核处理器系统中进行操作的时候，处理器能够嗅探其他处理器访问系统内存和它们内部缓存。处理器使用嗅探技术保证了它的内部缓存、系统缓存和其他处理器的缓存数据在总线上保持一致。如果通过嗅探一个处理器来来检测其他处理器打算写内存地址，而这个地址当前处于共享状态，那么正在嗅探的处理器会使它的缓存行无效，在下次访问相同内存地址的时候，强制执行缓存行填充。

为了提高处理速度，处理器不直接和内存进行通讯，而是先讲将系统内存数据督导内部缓存后再进行操作，**但是操作完不知道何时会写到内存**。如果对声明了 volatile 的变量进行写操作，JVM 就会向处理器发送一条 Lock 前缀的指令，将这个变量所在的缓存行的数据写回到系统内存。但是就算是写回到内存，如果其他处理器的缓存值还是旧的，再计算操作就会有问题。所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己的缓存值是不是过期了，当处理器发现自己的缓存行对应的内存地址被修改了，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行操作的时候，会重新从系统内存中把数据读到处理器缓存里。





#### 有序性

##### 基本概念

计算机在执行程序的时，为了提高性能，编译器和处理器尝尝会对指令做重排。

+ 单线程环境里面确保程序最终执行的结果和代码顺序执行的结果是一致的。
+ 处理器在进行重排的时候必须考虑指令之间的数据依赖性
+ 多线程换件中线程交替执行，由于编译器优化的存在，两个线程中使用的变量是否能保持一致性是无法确定，结果无法预测。

*在本线程内观察，所有操作都是有序的。一个线程观察另一个线程，所有操作都是无序的，无序是因为发生了指令重排序。在 Java 内存模型中，允许编译器和处理器对指令进行重排序，重排序不会影响单线程的执行，却会影响多线程并发执行的正确性。*

![三种重排](https://i.loli.net/2019/06/07/5cf9afcc9d88663363.png)

+ volatile 关键字通过添加内存屏障的方式来禁止指令重排，即重排序时不能把指令放到内存屏障之前。从而避免了多线程环境下出现程序乱序执行的现象。
+ synchronized 也可以保证有序性，它保证每个时刻只有一个线程执行同步代码，相当于让线程顺序执行同步代码。

##### 重排序1

![重排序1](https://i.loli.net/2019/07/18/5d307300dc62d84251.png)



>**Happen-before 规则**
>
>简单来讲如果 a happen-before b,  则 a 所做的任何操作对 b 是可见的。具体规则如下：
>
>1、程序次序规则：在一个单独的线程中，按照程序代码的执行流顺序，（时间上）先执行的操作happen—before（时间上）后执行的操作。
>2、管理锁定规则：一个unlock操作happen—before后面（时间上的先后顺序，下同）对同一个锁的lock操作。
>3、volatile变量规则：对一个volatile变量的写操作happen—before后面对该变量的读操作。
>4、线程启动规则：Thread对象的start（）方法happen—before此线程的每一个动作。
>5、线程终止规则：线程的所有操作都happen—before对此线程的终止检测，可以通过Thread.join（）方法结束、Thread.isAlive（）的返回值等手段检测到线程已经终止执行。
>6、线程中断规则：对线程interrupt（）方法的调用happen—before发生于被中断线程的代码检测到中断时事件的发生。
>7、对象终结规则：一个对象的初始化完成（构造函数执行结束）happen—before它的finalize（）方法的开始。
>8、传递性：如果操作A happen—before操作B，操作B happen—before操作C，那么可以得出A happen—before操作C。
>
>**内存屏障**
>
>是一个 CPU 指令，作用有两个：1. 保证特定操作的执行顺序；2. 保证某些变量的内存可见性（volatile的内存可见性即通过该特性实现的）。
>
>由于编译器和处理器都能执行重排优化。如果再指令间插入一条内存屏障则告诉编译器和 CPU，不管什么指令都不能和这条 Memory Barrier 指令重排序，也就是说 ==通过插入内存的屏障禁止在内存屏障前后的指令重排优化==。内存屏障另一个作用是强制刷出各种 CPU 的缓存数据，因此任何 CPU 上的线程都能读取到这些数据的最新版本。
>
>**volatile 写**
>
>![volatile 写](https://i.loli.net/2019/06/07/5cf9af95c34fc16296.png)
>
>**volatile 读**
>
>![volatile 读](https://i.loli.net/2019/06/07/5cf9afaf0959851529.png)
>
>1、 LoadLoad 屏障
>
>​	执行顺序： Load1 --> LoadLoad --> Load2
>
>​	确保 Load2 及其后续的Load指令执行前能访问到 Load1 的数据
>
>2、 StoreStore屏障
>
>​	执行顺序： Store1 --> StoreStore --> Store2
>
>​	确保 Store2 及其后的Store指令前执行能访问到 Store1 的数据
>
>3、 LoadStore屏障
>
>​	执行顺序： Load1 --> LoadStore --> Store2
>
>​	确保 Store2 及其后续的Store指令执行前能访问到 Load1 的数据
>
>4、 StoreLoad 屏障
>
>​	执行顺序： Store1--> StoreLoad --> Load2
>
>​	确保 Load2 及其后续的Load指令执行前能访问到 Store1的数据

##### 有序性2 

```java
public class VolatileDemo3 {
    int a = 0;
    boolean flag = false;
    
    void method1() {
        a = 1;
        flag = true;
    }
    
    void method2() {
        if (flag) {
            // 由于指令重排，method1 中的两条指令可能顺序变化。
            // 进而导致该处的 a 计算的值可能是 5 也可能是 6
            a += 5;
            flag = false;
        }
    }
}
```



### 缓存一致性

CPU 缓存的出现主要是为了解决 CPU 运算速度与内存读写速度不匹配的矛盾，因为 CPU 运算速度要比内存的读写速度快得多。现代 CPU 大多数情况不会直接访问内存，取而代之的是 CPU 缓存，CPU 缓存是位于 CPU 与内存之间的临时存储器。基本上可以分为 三级缓存：

1. 一级缓存：L1 Cache 位于 CPU 内核的旁边，是与 CPU 结合最紧密的 CPU 缓存。
2. 二级缓存:  L2 Cache 分内部和外部两种芯片，内部芯片二级缓存运行速度与主频相同，外部芯片二级内存运行速度只有主频的一半。
3. 三级缓存：L3 Cache 部分高端 CPU 才有

**每一级缓存所存储的数据全部都是下一级缓存的一部分。** CPU 读取数据的时候，首先从一级缓存中读取，再从 二级缓存中读取，最后从三级缓存中读取。每级缓存的命中率大概只有 80% 左右，20% 才从剩余的 二级和三级缓存中读取。

![CPU 读取缓存模式](https://i.loli.net/2019/06/07/5cf9af5e0209164560.png)

CPU 执行于是如下：

1. 程序以及数据被加载到主内存。
2. 指令和数据被加载到 CPU 缓存。
3. CPU 执行命令，把结果写到高速缓存。
4. 高速缓存中的数据写会到主内存。

多核心 CPU 可能会出现下面的情况：

1. 核心 1 读取了一个字节，根据局部性原理，它相邻的字节同样被读入核心 1 的缓存。
2. 核心 2 做了同样的操作，此时 核心 1 和 核心 2 的缓存拥有同样的数据。
3. 核心 1 修改了那个字节，被修改后，那个字节被写会到核心 1 的缓存中，但此时该信息并未写回主内存。
4. 核心 2 访问该字节，由于核心 1 并未将数据写回主内存，产生数据不同步。

多核心 CPU 缓存层次图如下：

![多核心 CPU 缓存层次图](https://i.loli.net/2019/06/07/5cf9b0738d61261003.png)



#### MESI 协议

MESI 是保持一致性的协议。方法是在 CPU 缓存中设置一个标志位，总共有四种状态：

1. **M**: Modify 修改缓存，当前 CPU 的缓存已经被修改，即与住内存中的数据已经不一致了。
2. **E**: Exclusive 独占缓存，当前 CPU 的缓存和内存中的数据保持一致，而且其他处理器并没有可使用的缓存数据。
3. **S**: Share 共享缓存，和内存保持一致的一份 Copy，多组缓存可以同时拥有针对同一地址的共享缓存段。
4. **I**： Invalid 失效缓存，说明 CPU 的缓存已经不可以再使用了。

读取的时候遵从以下几点：

1. 如果缓存状态是 I，那么就从内存中读取，否则就从缓存中直接读取。
2. 如果缓存处于 M 或 E 的CPU 读取到其他 CPU 有读操作，就把自己缓存写入到内存中，并将自己的状态设置为 S。
3. 只有缓存状态是 M 或 E 的时候， CPU 才可以修改缓存中的数据，修改后缓存状态变为 M。

MESI 协议中有两个执行成本比较大的行为：

1. 将某个 Cache Line 设置为 Invalid 状态。
2. 当某个 Cache Line 为 Invalid 状态写入新的数据。

CPU 是通过 Store Buffer 和 Invalidate Queue 组件来降低这两个操作的延时，细节下图所示：

![](https://i.loli.net/2019/06/07/5cf9b822e01ca79648.png)

当一个 CPU 进行写入的时候，首先会给其他 CPU 发送 Invalid 消息，然后把当前数据写入到 Store Buffer 中，然后异步在某个时刻写入到 Cache 中。当前 CPU 核如果要读取 Cache 中的数据，首先需要扫描 Store Buffer 之后再去读取 Cache。但是此时其他 CPU 看不到当前核的 Store Buffer 中的数据的，要等到 Store Buffer  中的数据刷到 Cache 之后才能触发失效操作。而当一个 CPU 核收到 Invalid 消息时，会吧 Invalid 消息写入到自身的 Invalidate Queue 中，然后异步的设置为 Invalid 状态。不过 CPU 使用 Cache 的时候并不会扫描 Invalidate Queue 部分，所以在极端的时间内出现脏读问题。**MESI 协议可以保证缓存一致性，但是无法保证实时性。** *CPU之间的可见性，即其他 CPU 可知，但并不是 实时可知*

#### 缓存一致性和内存一致性

**缓存一致性协议**是用来解决多个缓存副本之简数据一致性的问题，常用的用在 MESI 协议。

**内存一致性协议**则是用屏蔽 计算机硬件问题，主要解决并发编程中的原子性，有序性和一致性问题。保证并发环境下程序运行结果和预期结果是一致的。

*线程之间的内存可见性需要程序自身保证*

#### 缓存行

缓存行是 CPU Cache 中的最小单位，CPU Cache 是由若干缓存行组成，一个缓存行的大小通常是 64 字节，冰倩它有效地引用内存中的一块地址。一个 Java 的 long 类型是 8 个字节，因此在一个缓存行中可以存 8 个 long 类型的变量。

CPU Cache 和 缓存行示意图：

![CPU Cache 和 缓存行示意图](https://i.loli.net/2019/06/07/5cf9cfdb1d7b997469.png)

遍历一个长度为16的long数组data[16] 原始类型存在于主内存中，访问过程如下：

1. 访问 data[0]， CPU core 尝试访问 CPU Cache, 未命中
2. 尝试访问主内存，操作系统一次访问单位是一个 Cache Line 的大小即 64 个字节，这意味着从主内存卡获取到 data[0] 的值，同时将 data[0] - data[7] 加入到 CPU Cache 中。
3. 访问 data[1] - data[7] CPU  Core 尝试访问 CPU Cache 直接命中。
4. 访问 data[8], CPU  Core 尝试访问 CPU Cache 未命中。
5. 尝试访问主内存，重复步骤 2.

以上步骤可以看出 CPU 缓存在顺序访问内存数据的时候能够发挥最大的优势。

#### 伪共享

如果多个线程共享了同一个 CacheLine 那么任意一方修改操作都会使整个 CacheLine 失效，也就意味着 CPU 缓存将会彻底失效，降级为 CPU Core 和主内存的之间交互。

缓存行的解决方案是字节填充，只需要保证不同线程的变量存在于不同的 CacheLine 上即可。 在 java 8 中可以直接使用 `@Contended `注解，同时需要设置 JVM参数  `-XX:-RestrictContended=false` 

### 单例模式

##### DCL 单例模式

```java
private static SingleDemo instance;

public static SingleDemo getInstance() {
    if (instance == null) {
        synchronized (SingleDemo.class) {
            if (instance == null) {
                instance = new SingleDemo();
            }
        }
    }
    return instance;
}
```

DCL 双端检索不一定线程安全，原因是指令重排的存在，加入 volatile 可以禁止指令重排。

具体原因如下：

某一个线程执行到第一次检测，读取到 instance 不为 null  的时候, instance 引用的对象可能没有初始化完成。 因为 instance = new SingleDemo() 可以分为以下 3 步完成。

```java
memory = allocate();  // 1. 分配对象内存空间
instance(memory);     // 2. 初始化对象
instance = memory;    // 3. 设置 instance 指向刚分配内存的地址，
					  // 此时 instance != null
```

但是步骤2 和 3 不存在数据依赖关系，而且无论重拍前还是重拍后执行结果在单线程中并没有改变，因此这种重排是允许的。所以实际执行顺序可能如下：

```java
memory = allocate();  // 1. 分配对象内存空间
instance = memory;    // 3. 设置 instance 指向刚分配内存的地址，
					  // 此时 instance != null 
					  // 但是此时对象还没有初始化完成！
instance(memory);     // 2. 初始化对象
```

但是指令重排只会保证串行语义的一致性即单线程，并不会关心多线程的语义一致性，==所以当一个线程访问 instance != null 的时候，由于 instance 示例未必已经初始化完成，也就造成了线程安全问题。==

所以这里可以使用 `private static volatile SingleDemo instance;` 来禁止指令重排。

