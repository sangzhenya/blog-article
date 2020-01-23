## ChannelPipeline, ChannelHandler 和 ChannelHandlerContext

三者的关系是每当 ServerSocket 创建一个新的连接，就会创建一个 Socket 对应的就是目标客户端。每一个新创建的 Socket 都将会分配一个全新的 ChannelPipeline。每个 ChannelPipeline 包含多个 ChannelHandlerContext，其是一个双向链表，包装了通过 addLast 方法添加的 ChannelHandler。

![](http://img.programya.com/20200123231156.png)

ChannelSocket 和 ChannelPipeline 是一对一的关联关系，而 pipeline 内部的多个 Context 形成了链表，Context 只是对 Handler 的封装。当一个轻轻进来的时候，会进入 Socket 对应的 pipeline，并讲过 pipeline 中所有的 handler。

ChannelPipeline 继承了 inbound，outbound，iterable 等接口，表示其可以调用数据出入站的方法，同时也可以遍历内部的链表。

![](http://img.programya.com/20200123231335.png)