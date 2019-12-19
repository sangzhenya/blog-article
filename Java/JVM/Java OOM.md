## Java OOM

![Exception & Error](https://i.loli.net/2019/06/18/5d08fb6bd193464754.png)

### StackOverflowError

```java
public class StackOverFlowErrorDemo {
    public static void main(String[] args) {
        stackOverFlow();
    }

    private static void stackOverFlow() {
        stackOverFlow();
    }
}
```



递归调用过多，导致溢出。

```java
Exception in thread "main" java.lang.StackOverflowError
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
	at demo.oom.StackOverFlowErrorDemo.stackOverFlow(StackOverFlowErrorDemo.java:13)
```



### OutOfMemoryError: Java heap space

```java
public class JavaHeapSpaceDemo {
    public static void main(String[] args) {
        String string = "demo";
        while (true) {
            string += string + new Random().nextInt(100000000);
            string.intern();
        }
    }
}
```



内存不足导致，堆内存溢出

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:674)
	at java.lang.StringBuilder.append(StringBuilder.java:208)
	at demo.oom.JavaHeapSpaceDemo.main(JavaHeapSpaceDemo.java:13)
```



### OutOfMemoryError: GC overhead limit exceeded

JVM 参数： `-Xms50m -Xmx50m -XX:MaxDirectMemorySize=5m -XX:+PrintGCDetails`

```java
public class GCOverheadDemo {
    public static void main(String[] args) {
        int i = 0;
        List<String> list = new ArrayList<>();

        try {
            while (true) {
                list.add(String.valueOf(++i).intern());
            }
        } catch (Exception e) {
            System.out.println(i);
            e.printStackTrace();
            throw e;
        } finally {
        }
    }
}
```



GC 回收时间过长导致抛出 OutboundOfMemoryError，过长的定义是：超过 98% 的时间都用来做 GC 并且回收了不到了 2% 的内存。连续多次 GC 都只回收了不到 2% 的内存的极端情况下才抛出。如果不抛出 GC overhead limit  则会导致 GC 清理出来的内存很快被填满，迫使 GC 再次运行，这样形成恶性循环，CPU 使用率一直是 100%， 而 GC 没有任何成果。

```java
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.lang.Integer.toString(Integer.java:401)
	at java.lang.String.valueOf(String.java:3099)
	at demo.oom.GCOverheadDemo.main(GCOverheadDemo.java:17)
```



### OutOfMemoryError: Direct buffer memory

JVM 参数： `-Xms50m -Xmx50m -XX:MaxDirectMemorySize=5m -XX:+PrintGCDetails`

```java
public class DirectoryBufferMemoryDemo {
    public static void main(String[] args) {
        System.out.println("Config max direct memory:" + VM.maxDirectMemory() / 1024.0 / 1024.0 + "MB");
        try {
            Thread.sleep(3000);
        } catch (Exception e) {
        } finally {
        }
        ByteBuffer bb = ByteBuffer.allocateDirect(6 * 1024 * 1024);
    }
}
```



写 **NIO** 程序的时候经常使用 ByteBuffer 来读取或者写入数据，这是一种基于 Channel 与 Buffer的 I/O 方式，它可以直接使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆里面的 DirectByteBuffer 对象作为这块内存引用的进行操作。这样能在一些场景中显著的提高性能，因为避免了在 Java 堆和 Native 堆中来回复制数据。

`ByteFuffer.allocate(capability)` 是分配 JVM 堆内存，属于 GC 管辖范围，由于 COPY 所以速度相对较慢。

`ByteBuffer.allocateDirect(capability)`  是分配 OS 本地内存，不属于 GC 管辖范围，由于不需要内存 COPY 所以速度相对较快。

但是如果不断的分配本地内存，堆内存很少用，那么 JVM 就不需要执行 GC，DirectByteBuffer 对象们就不会被回收，这个时候如果堆内存充足，但是本地内存可能已经使用完了，则再次尝试分配本地内存就会出现 OutOfMemoryError, 程序崩溃。

```java
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
	at java.nio.Bits.reserveMemory(Bits.java:694)
	at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
	at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
	at demo.oom.DirectoryBufferMemoryDemo.main(DirectoryBufferMemoryDemo.java:19)
```



### OutOfMemoryError: unable to create new native thread

高并发请求服务器的时候，经常会出现此异常。准确的说该异常与对应的平台有关。

原因有两个：1 是应用创建了太多的线程了，一个应用创建多个现场，超过系统承载的极限。

2 是服务器不允许你的应用创建这么多线程， Linux 系统默认运行单个进程可以创建的线程数是1024个，如果应用创建了超过了这个数量，则会报此错误。

解决方案：1 是想办法降低应用的创建线程的数量，分析应用是否真的需要创建这么多线程，如果不是，则修改代码将线程数减低到最低。

2 对于有的应用，确实需要创建很多线程，远超过 Linux 系统默认的 1024 个线程的限制，可以通过 Linux 的服务器的配置，扩大 Linux 的默认限制。

```java
public class UnableCreateNativeThreadDemo {
    public static void main(String[] args) {
        for (int i = 0; ; i++) {
            System.out.println(i);
            new Thread(() -> {
                try {
                    Thread.sleep(Integer.MAX_VALUE);
                } catch (Exception e) {
                } finally {
                }
            }).start();

        }
    }
}

```

Linux 中设置文件如下：

![线程数配置](https://i.loli.net/2019/06/19/5d09160c745d655919.png)

### OutofMemoryError: Metaspace

JVM 参数： `-XX:MetaspaceSize=8m -XX:MaxMetaspaceSize=8m`

Java8  及以后版本使用 Metaspace 用来代替永久代。Metaspace 是方法区在 HotSpot 中的实现，它与永久代最大的区别在于：Metaspace 并不在虚拟机内存中，而是使用本地内存，也即在 java8 中， classese meatadata (the virtual machines inernal persentation of java class) 被存储在 叫做 Metaspace 的 native memory。永久代（在 Java 8 中被元空间 metaspace 取代了）存放了一下信息：

1. 虚拟机加载的类信息
2. 常量池
3. 静态变量
4. 即时编译后的代码
5. 模拟 Metaspace 空间溢出，就需要不断生成类往类空间中放，类占据空间总是超过 Metaspace 的指定的空间的大小。

```java
public class MetaspaceOOMDemo {
    static class OOMDemo{}
    public static void main(String[] args) {
        int i = 0;
        try {
            while (true) {
                i++;
                Enhancer enhancer = new Enhancer();
                enhancer.setSuperclass(OOMDemo.class);
                enhancer.setUseCache(false);
                enhancer.setCallback(new MethodInterceptor() {
                    @Override
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        return methodProxy.invoke(o, args);
                    }
                });
                enhancer.create();
            }
        } catch (Exception e) {
            System.out.println(i);
            e.printStackTrace();
        } finally {
        }
    }
}
```

```java
Caused by: java.lang.OutOfMemoryError: Metaspace
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	... 11 more
```

