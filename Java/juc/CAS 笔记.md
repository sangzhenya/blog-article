## CAS

> CAS ( Compare And Swap ) 比较并交换，是一条 CPU 并发原语。其作用是判断内存某个位置的值是否为预期值，如果是则更改为新值，这个过程是原子的。

### CAS

CAS 并发原语体现在 Java 预言中就是 sun.misc.Unsafe 类中各个方法。调用 Unsafe 类中的 CAS 方法， JVM 会帮我们实现出 CAS 汇编指令。这是一种依赖于硬件的功能，通过它实现了原子操作。由于 CAS 是一种传统原语，原语属于操作系统用语范畴，是由如果指令组成，用于完成某个功能的一个过程，==并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说 CAS 是一条 CPU 的原子指令，不会造成所谓的数据不一致问题==

### AtomicInteger

#### getAndIncrement 方法

```java
public final int getAndIncrement() {
    // this 当前对象； valueOffset 内存偏移量； +1
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

// 其中 unsafe 是类成员变量，如下所示：
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;
```

#### Unsafe

是 CAS 的核心类，由于 Java 方法无法直接访问底层系统，通过本地 ( native ) 方法来访问， Unsafe 相等于一个后门，基于该类可以直接操作特定内存的数据。 Unsafe 类存在于 sun.misc 包中，其内部方法块可以像 C 的指针一样直接操作内存，Java 中 CAS 操作的执行依赖于 Unsafe 类方法。==注意 Unsafe 类中所有方法都是 native 修饰的，也就是说 Unsafe 类中的方法都直接调用操作底层资源执行相应任务==

#### valueOffset

表示该变量值 在内存中偏移地址，因为 Unsafe 就是根据内存便宜地址获取数据的。

#### 变量 value 

被 volatile 修饰保证了多线程之间内存可见性



其中 Unsafe 类中的源码是：

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // var1 是当前对象； var2 是内存地址偏移量
        // 首先根据这两个值获取当前值
        var5 = this.getIntVolatile(var1, var2);
        // 1.：如果该地址上的值没有被修改，则该值等于 var5, 
        //    将该地址的值设置为新值，得到结果为 true， 
        //    取反后结果为 false。跳出循环。
        // 2.：如果该值已被修改，则该值不等于 var5 ，不会设置新值，同样得到结果
        //    为 false，取反结果为 true。进行下一次循环，重新获取值比较。
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
	
    return var5;
}
```

#### 过程如下

假设线程 A 和线程 B 两个线程同时执行 getAndAddInt 操作（分别跑在不同 CPU 上）

1. AtomicInteger 里面的 value 原始值为3，即主内存中的 AtomicInteger 的 value 为3，根据 JMM 模型，线程 A 和线程 B各自持有值为3的 value 的副本分别到各自的工作内存。
2. 线程 A 通过 getIntVolatile(var1, var2) 方法获取到 value3 的值为3，这时线程 A 被挂起。
3. 线程 B 也通过 getIntVolatile(var1, var2) 方法获取到 value3 值为3， 此时刚好线程 B 没有被挂起并执行 compareAndSwapInt 方法，比较内存值也为3，成功修改值为4，线程 B 结束。
4. 此时线程 A 回复，执行 compareAndSwapInt 方法比较，发现获取的值是3 和主内存中的值4不一致，说明该值已经被其他线程抢先一步修改过了，那么线程 A 本次修改失败，重新读取，再重新来一遍。
5. 线程 A 重新获取 value 值，因为 value 被 volatile 修饰，所以其他线程对它的修改是，线程 A总是能看到，线程 A继续执行 comareAndSwapInt 进行比较，直到成功。

### CAS 缺点

#### 循环开销时间大

doAndAddInt 方法执行时，有一个 do while 循环，如果 CAS 失败，则会一直进行尝试，如果 CAS长时间不成功，则可能会跟 CPU 带来很大的开销。

#### 只能保证一个共享变量的原子操作

当对一个共享变量操作的时候可以循环使用 CAS 的方法保证原子操作，但是对于多个共享变量操作的时候，循环 CAS 就无法保证操作的原子性，这个时候就需要用锁来保证原子性。

#### ABA 问题

CAS 会导致 ABA 问题，因为 CAS 算实现一个重要的前提是需要从内存中取出某个时刻的数据并且在当下时刻比较并替换，那么在这个时间差会导致数据变化，例如一个线程 1 从内存位置 V 中取出 A，这个时候另一个线程 2 也从内存中取出 A。并且线程2 进行了一些操作将值变成了 B，然后线程 2 又将 V 位置的数据变成 A，这个时候1 进行 CAS 操作，发现内存中仍然是 A，然后线程 1操作成功，尽管线程1 的 CAS 操作成功，但是不代表这个过程没有问题。

问题解决参考：

```java
public class ABADemo {
    static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);
    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        System.out.println("==========ABA================");
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);
        }).start();

        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2019) + "\t" + atomicReference.get());
        }).start();

        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("===============ABA SO =======");
        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t" + stamp);
            try {
                TimeUnit.SECONDS.sleep(1);

            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t" + atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t" + atomicStampedReference.getStamp());

        }, "t3").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t" + stamp);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean result = atomicStampedReference.compareAndSet(100, 101, stamp, stamp + 1);

            System.out.println(Thread.currentThread().getName() + "\t result::" + result);
            System.out.println(Thread.currentThread().getName() + "\t" +atomicStampedReference.getStamp());

        }, "t4").start();
    }
}

```



### 原子引用

示例代码如下：

```java
public class CASDemo {
    public static void main(String[] args) {
        User user1 = new User("zhangsan", 22);
        User user2 = new User("lisi", 25);

        AtomicReference<User> atomicReference = new AtomicReference<>();
        atomicReference.set(user1);
        System.out.println(atomicReference.compareAndSet(user1, user2) + "\t" + atomicReference.get().toString());
        System.out.println(atomicReference.compareAndSet(user1, user2) + "\t" + atomicReference.get().toString());

    }
}

class User{
    String userName;
    int age;

    public User(String userName, int age) {
        this.userName = userName;
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "userName='" + userName + '\'' +
                ", age=" + age +
                '}';
    }
}
```





