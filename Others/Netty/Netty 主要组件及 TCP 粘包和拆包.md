## Netty 

Netty 的主要组件有 Channel， EventLoop，ChannelFuture，ChannelHandler，ChannelPipeline 等。

### ChannelHandler

其中 ChannelHandler 是处理出入站数据的业务逻辑容器。例如通过实现 ChannelInboundHandler 接口，就可以接收入站事件和数据。当要给客户端端发送响应的时候，也可以从 ChannelInboundHandler 中写数据。业务逻辑通常写在一个或多个 ChannelInboundHandler 中。ChannelOutboundHanlder 类似，只是其用来处理出站数据的。

其中 ChannelPipeline 提供了 ChannelHandler  链的容器。以客户端应用为例，如果事件运动方向是从客户端到服务端，那么这个事件成为出站，即客户端发送给服务端的数据会经过 pipeline 中的一系列 ChannelOutboundHanlder，并且被这些 Handler 处理，反之则成为入站。

![ChannelPipeline](http://img.programya.com/20200119232204.png)

当 Netty 发送或者接收一个消息的时候就会发生一次数据转换。入站消息会被解码，即从字节转换为另外一种格式例如 Java 对象。如果是出站其则会被变成字节。Netty 提供了一系列使用的编解码器，他们都实现了ChannelInboundHandler 或者 ChannelOutboundHandler 接口。这些类中 channelRead 方法都已经被重写，以入站为例，对于每一个从入站 Channel 读取的消息，这个方法都会被调用，随后其将调用解码器所提供的 decode 方法进行解码，并将已经解码的字节转发给 ChannelPipeline 中的下一个 ChannelInboundHandler。

对于一个 MessageDecoder 来说每次入站都根据规则读取一定的字节数，将其解码成对应的格式。当没有更多的元素的时候，其内容会被发送到下一个 ChannelInboundHandler。

![Encoder & Decoder](http://img.programya.com/20200119234045.png)

编解码器都要会判断消息类型，只有和待处理的消息类型一致的时候才会调用编解码对应的方法，否则不会执行。此外在解码的时候需要判断缓存中的数据是否足够，否则接收到的结果和期望结果可能不一致。

### TCP 粘包和拆包

TCP 是面向连接的，面向流的，提供高可靠性服务。收发两端都要有一一相对的 Socket，因此，发送端为了将多个包更有效的发送给对方，其使用了优化算法（Nagle 算法）。将多次间隔较小且数据量小的数据，合并成一个大的数数据块，然后进行封包。这样做虽然提高了效率，但是接收端就难于分辨出完整的数据包了，因为面向流的通讯是无消息保护边界的。由于 TCP 无信息保护边界，需要在接收端处理消息边界问题，即粘包、拆包的问题。

![粘包拆包](http://img.programya.com/20200120220402.png)

例如客户端分别发送两个数据包 D1 和 D2 给服务端，由于服务端一次读取到的字节数是不确定的，谷可能存上图中的四种情况。

1. 服务端分两次读取两个独立的数据包，分别是 D1 和 D2，没有粘包和拆包。
2. 服务端一次接收两个数据包，D1 和 D2 粘合在一起，称为 TCP 粘包。
3. 服务端分两次读取到了数据包，地刺读取到了完整的 D1 包和 D2 包的部分内容，第二次读取到了 D2 包剩余的内容，这称之为 TCP 拆包。
4. 服务端分两次读取到了数据包，第一次读取到 D1 包的部分内容，第二次读取到了 D1 包的剩余内容和 D2 包。

TCP 的粘包和拆包解决方案即使用自定义协议和编解码器解决，关键是要解决服务端每次读取数据长度的问题，这个问题解决就不会出现服务器多读和少读数据的问题，从而避免 TCP 粘包和拆包。





