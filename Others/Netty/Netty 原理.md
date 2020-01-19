## Netty 

Netty 的主要组件有 Channel， EventLoop，ChannelFuture，ChannelHandler，ChannelPipeline 等。

其中 ChannelHandler 是处理出入站数据的业务逻辑容器。例如通过实现 ChannelInboundHandler 接口，就可以接收入站事件和数据。当要给客户端端发送响应的时候，也可以从 ChannelInboundHandler 中写数据。业务逻辑通常写在一个或多个 ChannelInboundHandler 中。ChannelOutboundHanlder 类似，只是其用来处理出站数据的。

其中 ChannelPipeline 提供了 ChannelHandler  链的容器。以客户端应用为例，如果事件运动方向是从客户端到服务端，那么这个事件成为出站，即客户端发送给服务端的数据会经过 pipeline 中的一系列 ChannelOutboundHanlder，并且被这些 Handler 处理，反之则成为入站。

![ChannelPipeline](http://img.programya.com/20200119232204.png)

当 Netty 发送或者接收一个消息的时候就会发生一次数据转换。入站消息会被解码，即从字节转换为另外一种格式例如 Java 对象。如果是出站其则会被变成字节。Netty 提供了一系列使用的编解码器，他们都实现了ChannelInboundHandler 或者 ChannelOutboundHandler 接口。这些类中 channelRead 方法都已经被重写，以入站为例，对于每一个从入站 Channel 读取的消息，这个方法都会被调用，随后其将调用解码器所提供的 decode 方法进行解码，并将已经解码的字节转发给 ChannelPipeline 中的下一个 ChannelInboundHandler。

对于一个 MessageDecoder 来说每次入站都根据规则读取一定的字节数，将其解码成对应的格式。当没有更多的元素的时候，其内容会被发送到下一个 ChannelInboundHandler。

![Encoder & Decoder](http://img.programya.com/20200119234045.png)

编解码器都要会判断消息类型，只有和待处理的消息类型一致的时候才会调用编解码对应的方法，否则不会执行。此外在解码的时候需要判断缓存中的数据是否足够，否则接收到的结果和期望结果可能不一致。





