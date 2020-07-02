## RabbitMQ

[toc]

### 概述

多数应用中可以通过消息服务中间件来提升系统异步通信、扩展解耦的能力。其有两个重要的概念：消息代理和目的地。当消息发送者发送消息后，将由消息代理接管，消息代理保证消息传递到指定的目的地。

主要有两种形式：

1. 队列 Queue，点对点通讯。消息发送者发送消息，消息代理将其放入到一个队列中，消息接受者从队列中获取消息后消息移除队列。消息只能有唯一的发送者和接收者，但是不是不是说一个队列只能有一个接收者。
2. 主题 Topic：发布订阅模式。发送者将消息发送到主题，多个接收者监听该主题，如果有消息到达的时候多个接收者均会接收到消息。

两种 Message 规范：

1. JMS 即 Java  Message Service, Java 消息服务，基于 JVM 消息代理的规范，有 ActiveMQ 和 HornetMQ 等实现。仅支持 Queue 和 Topic 两种 Model 类型，其支持多种消息类型，例如 TextMessage, MapMessage, BytesMessage, StreamMessage, ObjectMessage, Message 等等。但是不支持跨语言和平台。
2. AMOP 即 Advanced Message Queuing Protocol，高级消息队列协议，也是一个消息代理规范，兼容 JMS，RabbitMQ 是 AMQP 的实现。其支持 Direct Exchange，Fanout Exchange, Topic Exchange, Header Exchange, System Exchange 五种类型，但是本质上来说和 Topic类型没有太大差别，对于消息类型仅支持 byte[] 类型，需要的时候可以将消息序列化后发送出去，另外 AMOP 支持跨语言和平台。

### Spring 对 Message 的支持

#### Spring 对 JMS 的支持

Spring 通过 spring-jms 支持 JMS，通过 ConnectFactory 实现消息的连接，其提供了 JSMTemplate 用于操作消息，同时支持通过 `@JmsListener` 注解监听消息代理发布的 Message，可以通过 `@EnableJms` 开启支持。



#### Spring 对 AMOP 的支持

Spring 通过 spring-rabbit 提供了对于 AMOP 的支持，同样通过 ConnectFacotry 实现消息的连接，使用 RabbitTemplate 来发送消息，支持通过 `@RabbitListener`  的方式监听消息代理发布的消息，可以通过 `@EnableRabbit` 开启支持。



### Rabbit MQ 基本

Rabbit MQ 是一个由 erlang 开发的 AMQP 的开源实现，主要有以下几个核心概念：

1. Message, 消息是不具名的，其由消息头和消息体组成。消息体是不透明的，而消息头则是由一系列可选的属性组成，这些属性包括 routing-key 路由键，priority 优先权，delivery-mode 持久性存储等。
2. Publisher, 消息的生产者，也是一个向交换器发布消息的客户端程序。
3. Exchange，交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。Exchange 有四种类似：Direct，Fanout，Topic 和 Headers，不同类型的 Exchange 转发消息的策略不同。
4. Queue, 消息队列，用来保存消息直到发送给消费者。其实消息的容器，也是消息的终点。一个消息可投入一个或者多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。
5. Binding，绑定，用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以将交换器理解成一个绑定构成的路由表。Exchange 和 Queue 的绑定可以是多对多的关系。
6. Connection，网络连接，例如一个 TCP 连接。
7. Channel，信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的 TCP 连接内的虚拟连接，AMQP 命令都是通过信道发送出去的，不管是发布消息，订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁 TCP 都是非常昂贵的开销，所以引入了信道的概念，以复用一条 TCP 连接。
8. Consumer 消息消费者，表示一个从消息队列中取得消息的客户端应用程序。
9. Virtual Host 虚拟主机，表示一批交换器、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个 vhost 本质上就是一个mini 的 RabbitMQ 的服务器，用于自己的队列，交换器，绑定和权限机制。vhost 是 AMQP 概念的基础，必须在连接的时候指定。RabbitMQ 默认 vhost 是 /.
10. Broker 表示消息队列服务器实体。

结构图如下：

![结构图](https://i.loli.net/2020/07/02/7Rpt26EzYDixM3j.png)

