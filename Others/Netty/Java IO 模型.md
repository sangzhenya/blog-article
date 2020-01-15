## Java IO 模型

Java 模型简单的理解就是使用什么样的通道惊醒数据的发送和接收，这很大程度上决定了程序通讯的性能。Java 支持 3 种网络编程模型 IO：BIO，NIO，AIO。

**BIO** 是同步阻塞模型，服务器实现模式为一个连接对应一个线程，即客户端有请求的时候服务端就需要启动一个线程进行处理，如果这个连接不作任何事情会造成不必要的开销。其适用于连接数目较小且固定的架构，这种方式对服务器资源要求比较高并发局限于应用中。参考立刻 [Code](https://github.com/sangzhenya/code-samples/blob/master/inetty/src/main/java/com/xinyue/inetty/iio/BIOMain.java)

![BIO 模型](http://img.programya.com/20200112193203.png)

**NIO** 是同步非阻塞模型，服务器实现模式为一个线程处理多个请求连接，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有 IO 请求就进行处理。其适用于连接数目较多且连接比较短的架构，轻操作，比如聊天服务器，弹幕系统，服务器通讯等。

![NIO 模型](http://img.programya.com/20200112193556.png)

**AIO** 是异步非阻塞模型，引入了异步通道的概念，采用了 Proactor 模型，简化程序的编写，有效的请求才会启动线程，特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用。其适用于连接数目较多且连接较长的架构，重操作，例如相册服务器等。

#### Java NIO

NIO 同步非阻塞模型主要有三个核心组件： Channel，Buffer，Selector。其是面向缓冲区或者面向块编程的。数据读取到一个它稍后处理的缓冲区，需要时从缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络。流程如下：

![NIO](http://img.programya.com/20200112223957.png)



通俗来说NIO 可以做到一个线程来处理多个操作，假设有10000 个请求过来，根据实际情况，可以分配 50 个或者 100 个线程来处理。HTTP2.0 使用了多路复用技术，做到同一个连接并发处理多个请求，而且并发请求的数量比 HTTP1.1 大几个数量级。

**NIO 和 BIO 比较：**BIO 以流的方式处理数据，而 NIO 以块的方式处理数据，块 I/O 的效率比流 I/O 高很多。BIO 是阻塞的，NIO 则是非阻塞的。BIO 基于字节流和字符流进行操作，而 NIO 基于 Channel，Buffer 进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector 用于监听多个通道的事件（例如连接请求，数据到达等），因此使用单个线程就可以监听多个客户端通道。

#### Channel，Buffer，Selector

**Channel，Buffer，Selector关系：**每个 Channel 对应一个 Buffer，多个 Channel 可以注册到一个 Channel 上，每个Selector 对应一个线程。切换到哪个 Channel 是根据 Event 决定的，Event 就是一个重要的概念，Selector 会根据不同的 Event 在各个 Channel 上切换。Buffer 就是一个内存块，底层是一个数组。数据的读取写入都是通过 Buffer，BIO 中的流是单向的的，但是 NIO 的 Buffer 是双向的，既可以读也可以写可以通过 flip 方法切换。Channel 也是双向的，可以返回底层操作系统的情况。

#### Buffer

Buffer 即缓冲区，本质上就是一个可以读写的内存块，可以理解成一个容器对象，该对象提供了一组方法，可以更轻松的使用内存块，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。Channel 提供从文件、网络读取数据的渠道，但是读取或者写入数据都必须经过 Buffer。

![存储过程](http://img.programya.com/20200113194358.png)

在 NIO 中 Buffer 是一个顶层父类，其实一个抽象类。常用的有 `ByteBuffer`, `ShortBuffer`， `CharBuffer`, `IntBuffer`, `LongBuffer`, `DoubleBuffer`,`FloatBuffer`。Buffer 类定义了所有缓冲区都具有的四个属性来提供关于其所包含的数据信息：

```java
// 标记，书签。
private int mark = -1;
// 下一个要被读或者写的元素的索引，每次读写缓冲区都会修改该值。为下次读写做准备。
private int position = 0;
// 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。其可以修改。
private int limit;
// 容量，即可以容纳最大的数据量；在缓冲区创建时被设定且不能改变。
private int capacity;
```

几个常用方法如下：

```java
// 获取最大容量
public final int capacity();
// 获取当前的位置
public final int position();
// 设置当前的位置
public Buffer position(int newPosition);
// 返回极限值
public final int limit();
// 设置极限值
public Buffer limit(int newLimit);
// 清空缓冲区
public Buffer clear();
// 翻转缓冲区
public Buffer flip();
// 缓冲区中是否还有剩余元素
public final boolean hasRemaining();
// 该缓冲区是否是只读的
public abstract boolean isReadOnly();
// 该缓冲区是否有具有可访问的底层实现数组。
public abstract boolean hasArray();
// 返回此缓冲区的底层实现数组
public abstract Object array();
```

**ByteBuffer** 是最常用的，主要方法如下：

```java
// 创建直接缓冲区
public static ByteBuffer allocate(int capacity);
// 创建直接缓冲区
public static ByteBuffer allocateDirect(int capacity);
// 获取当前位置的元素
public abstract byte get();
// 获取指定位置的元素
public abstract byte get(int index);
// 从当前位置添加元素
public abstract ByteBuffer put(byte b);
// 在指定位置放置元素
public abstract ByteBuffer put(int index, byte b);
```

#### Channel

Channel 即通道，类似于流，但其可以同时进行读写，可以实现异步读写数据，可以从缓冲区读取数据也可以写数据到缓冲区中。常用的 Channel 类有 `FileChannel`  文件读写, `DatagramChannel` UDP 读写, `ServerSocketChannel`, `SocketChannel` TCP 读写。其中 `FileChannel` 是对本地文件进行 IO 操作，常用方法有：

```java
// 从 Channel 中读取数据并放到缓冲区中
public abstract int read(ByteBuffer dst) throws IOException;
// 把缓冲区的数据写到 Channel 中
public abstract int write(ByteBuffer src) throws IOException;
// 从目标 Channel 中复制数据到当前 Channel
public abstract long transferFrom(ReadableByteChannel src, long position, long count);
// 把数据从当前 Channel 复制到目标 Channel
public abstract long transferTo(long position, long count, WritableByteChannel target);
```

ByteBuffer 支持类型化的 put 和 get，put 的数据类型要和 get 使用的数据类型一致。否则则会抛出 `BufferUnderflowException` 异常。还可以将一个普通的 Buffer 转成只读 Buffer。NIO 还提供了 MappedByteBuffer 可以让文件直接在内存中进行修改（堆外内存），而如果同步到文件由 NIO 来完成。此外 NIO 还支持通过多个 Buffer 完成读写操作，即 Scattering 和 Gathering。

![NIO 流程](http://img.programya.com/20200113233606.png)

#### Selector

Java 的 NIO 用阻塞式 IO 方法。如果用一个线程，处理多个客户端的连接就会使用到 Selector。Selector 能够检测多个注册的 Channel 上是否有 Event。多个 Channel 是以事件的方式注册到同一个 Selector。如果有 Event 则获取  Event 然后针对每个事件进行相应的处理。这样就可以只用一个线程管理多个 Channel。通过 Selector 只有在真正有读写 Event 时才会进行读写，所以就可以很大程度上减少系统的开销，并且不必为每个链接创建一个线程，不用去维护多线程，也避免了多线程之间的上下文切换的开销。

Selector 类是一个抽象类，常用方法如下：

```java
// 获取一个选择器对象
public static Selector open();
// 监控所有注册的 Channel，当其中有 IO 操作的时候将对应的 SelectionKey 加入到内部集合
// 并返回，通过参数可以设置超时时间
public abstract int select(long timeout);
public abstract int select();
// 不阻塞
public abstract int selectNow() throws IOException;
// 从内部集合中的得到所有的 SelectionKey
public abstract Set<SelectionKey> selectedKeys();
// 唤醒 Selector
public abstract Selector wakeup();
```

Selector，SelectionKey，ServerSocketChannel 和 SocketChannel 关系图如下：

![关系图](http://img.programya.com/20200114213632.png)

SelectionKey 表示 Selector 和 SocketChannel 的注册关系，公有四种：

1. OP_READ：有写操作，值为1
2. OP_WRITE：有写操作，值为 4
3. OP_CONNECT：连接已经建立，值为 8
4. OP_ACCEPT：有新的 Client 连接，值为 16

主要方法有：

```java
// 得到与之关联的 Selector
public abstract Selector selector();
// 得到与之关联的 Channel
public abstract SelectableChannel channel();
// 得到与之关联的共享数据
public final Object attachment();
// 设置或改变监听事件
public abstract SelectionKey interestOps(int ops);
// 是否可以 accept
public final boolean isAcceptable();
// 是否可读
public final boolean isReadable();
// 是否可以写
public final boolean isWritable();
```

ServerSocketChannel 在服务端监听新的客户端  Socket 连接，主要方法如下：

```java
// 得到一个 ServerSocketChannel 通道
public static ServerSocketChannel open();
// 设置服务器端口号
public final ServerSocketChannel bind(SocketAddress local);
// 设置阻塞或非阻塞模式，取值 false 表示采用非阻塞模式
public final SelectableChannel configureBlocking(boolean block);
// 接受一个连接，返回代表这个连接的通道对象
public abstract SocketChannel accept();
// 注册一个选择器并设置监听事件
public final SelectionKey register(Selector sel, int ops);
```

SocketChannel 网络 IO 通道，负责具体读写操作。NIO 把 Buffer 的数据写入到 Channel，或者把 Channel 里的数据读到 Buffer 中。主要方法如下：

```java
// 得到一个 SocketChannel
public static SocketChannel open();
// 设置阻塞或非阻塞模式
public final SelectableChannel configureBlocking(boolean block);
// 连接服务器
public abstract boolean connect(SocketAddress remote);
// 连接失败的时候，可以通过该方法完成操作
public abstract boolean finishConnect();
// 往 Channel 中写数据
public abstract int write(ByteBuffer src);
// 从 Channel 中读数据
public abstract int read(ByteBuffer dst);
// 注册一个 Selector 并设置监听事件，最后一个参数可以设置共享数据。
public abstract SelectionKey register(Selector sel, int ops, Object att);
// 关闭通道
public final void close();
```

#### 零拷贝

零拷贝是网络编程的关键，Java 程序中，常用的零拷贝有 mmap 内存映射和 sendFile 两种方式。以下分别是传统 IO。mmap 映射 和 sendFile 的 copy 过程图：

传统 IO

![传统 IO](http://img.programya.com/20200115204611.png)

首先是从硬盘经过 DMA 拷贝 到 kernel buffer，然后从kernel buffer 经过cpu 拷贝到 user buffer ,比如拷贝到应用程序，然后从user buffer 拷贝到 socket buffer ，最后从socket buffer 拷贝到 protocol engine 协议栈，四个步骤。经过了用户态到内核态，内核态到用户态，用户态到内核态，最后由内核态回到用户态总共四次切换。

Mmap 映射

![mmap 映射](http://img.programya.com/20200115204358.png)



mmap 通过内存映射，将文件映射到内核缓冲区，同时用户空间可以共享内核空间的数据。只需要从内核缓冲区拷贝到 Socket 缓冲区即可，这将减少一次内存拷贝（从 4 次变成了 3 次），但不减少上下文切换次数。但是不减少上下文切换的次数。

sendFile

![sendFile](http://img.programya.com/20200115204744.png)

sendFile 的方式数据根本不经过用户态，直接从内核缓冲区进入到 Socket Buffer，由于和用户态完全无关，就减少了一次上下文切换。数据被 DMA 引擎从文件复制到内核缓冲区，然后调用，然后掉一共 write 方法时，从内核缓冲区进入到 Socket。在 Linux 2.4 中对 sendFile 进行了优化，避免了从内核缓冲区拷贝到 SocketBuff 的操作而是直接拷贝到协议栈，从而减少一次 Copy，如下所示：

![sendFile](http://img.programya.com/20200115211703.png)

mmap 和 sendFile 的区别如下：

1. mmap 适合小数据量的读写，sendFile 适合大文件传输
2. mmap 需要 4 次上下文切换，3 次数据拷贝；sendFile 需要 3 次上下文切换，最少需要 2 次数据拷贝。
3. sendFile 使用 DMA 方式减少了 CPU 拷贝，mmap 则不能减少。

所谓的零拷贝是从操作系统的角度来说，因为内核缓冲区之间没有数据是重复的，即只有 kernel buffer 一份数据。零拷贝能带来更少的数据复制还能带来其他的性能优势，例如更少的数据切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算。

#### Java  AIO

JDK 1.7 中引入了 Asynchronus I/O 即 AIO。在 IO 编程中，常用到两种模式：Reactor 和 Proactor。Java 的 NIO 就是 Reactor，当有事件触发的时候，服务器端得到通过，进行相应的处理。AIO 即 NIO 2.0 是异步不阻塞 IO，AIO 引入异步通道的概念，采用了Proactor 模式，简化了程序编写，有效的请求才启动线程，其特点是先由操作系统完成后才通知服务端程序启动线程处理，一般用于连接数较多且连接时间较长的应用。