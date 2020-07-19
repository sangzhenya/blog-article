---
title: "Rabbit MQ"
tags: ["Rabbit MQ"]
categories: ["MQ"]
date: "2019-06-03T09:00:00+08:00"
---

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
9. Virtual Host 虚拟主机，表示一批交换器、消息队列和相关对象。类似数据库的一个 DB。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个 vhost 本质上就是一个mini 的 RabbitMQ 的服务器，用于自己的队列，交换器，绑定和权限机制。vhost 是 AMQP 概念的基础，必须在连接的时候指定。RabbitMQ 默认 vhost 是 /。
10. Broker 表示消息队列服务器实体。

结构图如下：

![结构图](https://i.loli.net/2020/07/02/7Rpt26EzYDixM3j.png)

RabbitMQ 启动后默认使用两个端口，一个 15672 作为 Web 管理端口，一个是 5672 作为通讯端口。

### Rabbit MQ 工作模式

Rabbit MQ 可以不使用 Exchange 而直接将 Message 发送到 Queue，Consumer 从 Queue 中读取数据，就是简单的消息队列模式，当然一个 Queue 可以设置一个或者多个 Consumer，相同 Queue 直接的 Consumer 是竞争关系。

另外看一下 AMQP 中的消息路由，其和 Java 开发者熟悉的 JMS 存在一些差别，AMQP 中增加了 Exchange 和 Binding 的校色。生产中把消息发布到 Exchange上，消息最终到达队列并被消费者接收，而 Binding 决定交换器的消息应该发送到哪个队列。结构如下：

![结构如下](https://i.loli.net/2020/07/04/mGzcrdZAPMaCFgU.png)

#### Exchange 的类型

主要有四种：Direct、Fanout、Topic 和 Headers。

Direct Exchange：消息中的路由键（Routing Key） 如果和 Binding 队列中的 Binding Key 一致的话，交换器就将消息发到对应的队列中。路由键与队列名完全匹配，如果一个队列绑定到交换器要求路由键为 Dog 那么仅转发 Routing Key 为 Dog 的消息，不会转发其他路由键的消息，例如 dog.puppy，是一个简单的完全匹配的模式。

![Direct Exchange](https://i.loli.net/2020/07/04/cJ1LOaRrt5GBgjC.png)

Fanout Exchange：每个发到 Fanout Exchange 的消息都会分到所有绑定的队列上。Fanout 交换器不处理 Routing Key 而是简单的将队列绑定到 Exchange 上，每个发送到交换器的消息都会被转发到与该交换器绑定的所有队列上。和子网广播类似，每台子网内的主机都会受到一份复制消息，Fanout 类库转发消息最快。

![Fanout](https://i.loli.net/2020/07/04/9SQLP8DqTWfAnbj.png)

Topic Exchange：其通过模式匹配分配消息，可以将 Routing Key 和某个模式进行匹配，此时对象需要绑定到一个模式上。它将 Routing Key 和 Binding Key 的单子切分成单词，这些单子之间使用 `.` 分割。同样识别两个通配符，`#` 表示 0 个或多个单词，`*` 表示匹配一个单词。

![Topic Exchange](https://i.loli.net/2020/07/04/dDxSAmW9IFM6nf8.png)

Headers Exchange：其匹配的是 AMQP 消息的 Header，而不是使用路由键，Headers Exchange 和 Direct Exchange 几乎一致，但是性能却差很多，所以一般情况使用不到。


#### 生产者和消费者流转过程

生产者流转过程：

1. 客户端与代理服务器 Broker 建立连接。会调用 `newConnection` 方法，该方法会封装 Protocol Header 0-9-1 的报文，以此通知 Broker 在本次交互采用的是 AMQPO-9-1 协议，然后 Broker 返回  `Connection.Start` 来建立连接。
2. 客户端调用 `connect.createChannel` 方法，此方法开启信道，其包装的 `Channel.open` 命令发送给 Broker，等待 `Channel.basicPublish` 方法，对应的 AMQP 命令为 `Basic.Publish` 这个命令包含 Content Header 和 Content Body 两部分，Content Header 包含消息体属性：投递模式，优先级等。Content Body 就是消息体。
3. 客户端发送完消息需要关闭资源的时候，设计到 `Channel.Close`, `Channel.Close-Ok`, `Connection.Close`, `Connection.Close-Ok` 的命令交互。

流程图如下：

![image-20200718090606068](https://i.loli.net/2020/07/18/dyagoKul3npZ6DU.png)

消费者流转过程：

1. 客户端与代理服务器 Broker 建立连接。与生产者一样
2. 客户端开启信道。与生产者一样。
3. 在真正消息之前，消费者客户端需要向 Broker 发送 `Basic.Consume` 方法，将 Channel 设置为接收模式，之后 Broker 回执 `Basic.Consume-OK` 告诉消费者准备好接收消息。
4. Broker 向消费者推送消息，即 `Basic.Deliver` 命令，这个命令和 `Basic.Publish` 命令一样会带有 Content Header  和 Content  Body。
5. 消费者接收并正确消费之后，向 Broker 发送确认 `Basic.ACK` 命令。
6. 客户端接收完消息关闭资源。与生产者一样。

流程图如下：

![image-20200718090745729](https://i.loli.net/2020/07/18/UvsDzNfILuhn1T9.png)



### RabbitMQ 的几个概念

#### 过期时间

RabbitMQ 可以对消息设置过期时间 TTL ，在这个时间内可以被消费者消费，过期之后自动删除。Rabbit MQ 可以通过两种方式设置 TTL：

1. 设置 Queue 属性设置，则 Queue 中所有的消息都有相同的 TTL。
2. 对消息进行单独设置，每条消息的 TTL 可以不同。

在 Web 控制台中新建 Queue 的时候可以直接设置参数如下：

![image-20200718112330186](https://i.loli.net/2020/07/18/U84eEWuVAnQJ7gi.png)



通过代码声明 Queue 的时候可以使用如下：

```java
Queue queue = new Queue("demo-ttl", true, false, false, Collections.singletonMap("x-message-ttl", 12000));
amqpAdmin.declareQueue(queue);
```

对于单独 Message 设置可以通过 MessageProperties 的 expiration 属性设置，如下：

```java
MessageProperties messageProperties = new MessageProperties();
messageProperties.setExpiration("12000");
```

上面 TTL 的单位都是毫秒，如果同时指定了 Queue 与 Message 的 TTL，两者中最小的那个起作用。

#### 死信队列

首先看一下 DLX (Dead Letter Exchange) 即死信交换机。当有一个消息变为死信的时候，其能被重新发送到另外一个交换机中。这交换机就是 DLX，绑定 DLX 的队列被称为死信队列。消息变为死信的原因有以下几种可能：消息被拒绝；消息过期；队列达到最大长度。

DLX 也是一个正常的交换机，和一般的交换机没有区别，其能在任何队列上被指定。当这个队列存在死信的时候，RabbitMQ 就会自动将这个消息重新发布到设置的 DLX 上去，进而被路由到另外一个队列中（死信队列）。

在 Web 控制台设置如下：

首先有一个正常的 Exchange 例如 **demo.direct**，一个 DLX 例如： **demo.dlx**。然后一个正常的 Queue 例如：**demo.news.queue**，在创建该 Queue 的时候设置 `x-dead-letter-exchange` 属性指向 DLX。如下图所示：

![image-20200718115747273](https://i.loli.net/2020/07/18/Ogy7MtLHhZ4NeTE.png)

然后需要一个死信队列例如：**demo.news.dlx**。并使用 `demo-ttl-dlx` 绑定 key 绑定到 DLX (demo.dlx)。然后将正常的 Exchange (demo.direct) 和 正常的 Queue (demo.news.queue) 使用 `demo-ttl-dlx` 绑定 key 绑定。

然后向 demo.direct 上发送路由 key 为 `demo-ttl-dlx` 的 Message 并设置 TTL，在 TTL 的时间范围内可以看到在 demo.news.queue 上存在该消息。当超过了 TTL 之后可以看到该消息已经被发送到了 demo.news.dlx 上。

![image-20200718121118663](https://i.loli.net/2020/07/18/1JaEzft2xoIpPrS.png)

这里和正常创建唯一的不同即是通过 properties 指定死信队列的位置，在代码中同样也是通过配置属性指定：

```java
Queue queue = new Queue("demo-news.queue", true, false, false, Collections.singletonMap("x-dead-letter-exchange", "demo.dlx"));
amqpAdmin.declareQueue(queue);
```

#### 延迟队列

延迟队列顾名思义即存储延迟消息的队列，即消息发送出去以后不想立刻被消费者拿到，而是等待特定的时间后，消费者才能拿到这个消息进行消费。在 RabbitMQ 没有直接提供这样的方式，但是可以通过 TTL + DLX 的方式实现，为正常 Queue（或者说中转 Queue）设置一个 TTL，然后设置其 DLX，在 DLX 上绑定到真正消费消息的 Queue 上。这样就实现了延迟队列。

#### 消息确认

确保消息被送达有两种方式：发布确认和事务。但是二者只能选择一个，如果使用事务则不能使用确认模式，反之亦然。

##### 使用消息确认

如果要启用消息确认可以在配置文件中配置：

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated # 是否到 Broker Callback
    publisher-returns: true # 是否到 Queue Callback
```

然后可以在 `rabbitTemplate` 设置相应的 `callback` 如下所示：

```java
String demoRoutingKey = "usa.news";
String demoExchange = "demo.direct";
byte[] messageBody = new ObjectMapper().writeValueAsBytes(Collections.singletonMap("id", 1));
MessageProperties messageProperties = new MessageProperties();
messageProperties.setReceivedRoutingKey(demoRoutingKey);
messageProperties.setReceivedExchange(demoExchange);
Message sendMessage = new Message(messageBody, messageProperties);
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
  log.info(ack);
  log.info(cause);
  log.info(correlationData);
});
rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
  log.info(message);
  log.info(replyCode);
  log.info(replyText);
  log.info(exchange);
  log.info(routingKey);
});
// 如果配置了 publisher-returns 且没有配置 mandatory 为 false 则默认 mandatory 为 true
//rabbitTemplate.setMandatory(true);
CorrelationData correlationData = new CorrelationData();
correlationData.setId(UUID.randomUUID().toString());
correlationData.setReturnedMessage(sendMessage);
rabbitTemplate.send(demoExchange, demoRoutingKey, sendMessage, correlationData);
```

#####  使用事务

如果业务处理伴随着消息的发送，那么业务处理失败后，事务回滚，且消息不发送出去。简单示例代码如下：

```java
@Configuration
public class RabbitConfig {
  	// 引入事务管理器
    @Bean
    public TransactionManager transactionManager(ConnectionFactory connectionFactory) {
        return new RabbitTransactionManager(connectionFactory);
    }
}
```

```java
// 标注以事务模式发送消息
@Transactional(rollbackFor = Exception.class)
public void testTransacted() {
  rabbitTemplate.setChannelTransacted(true);
  rabbitTemplate.convertAndSend("demo.direct", "usa.news", "12123");
  //        int a = 1 / 0;
}
```

如果使用了事务那么在发送之前会调用 Channel 的 `txSelect` 方法主要是将 Channel 设置成事务模式，事务成功后会调用 `txCommit` 提交事务，失败则调用 `txRollback` 回滚事务。

#### 消息追踪

消息中心的消息追踪需要通过 Trace 实现，Trace 是 RabbitMQ 用于记录每一次发送的消息，方便使用 RabbitMQ 开发调试。可通过插件形式提供可视化解码。Trace 启动后自动会创建 amq.rabbitmq.trace Exchange，每个队列都会自动绑定该 Exchange，绑定后发送给队列的消息都会记录到 Trace 日志中。

常用的命令如下：

1. rabbitmq-plugins list
2. rabbitmq-plugins enable rabbitmq_tracing
3. rabbitmq-plugins disable rabbitmq_tracing
4. rabbitmqctl trace_on
5. rabbitmqctl trace_off
6. rabbitmqctl set_user_tags guest administrator

在 enable 的 rabbitmq_tracing 后，在 Web 管理的 admin 界面就可以看到了 tracing tab。

![image-20200718163525694](https://i.loli.net/2020/07/18/xEYOU9cKrwIe1XJ.png)

然后可以添加一个 Trace 

![image-20200718163550603](https://i.loli.net/2020/07/18/72LjJ8g4GKwlQqF.png)

然后就会显示所有 Trace 了：

![image-20200718163608828](https://i.loli.net/2020/07/18/3FoM54xTJcYpUag.png)

点击 demo-trace.log 便可以查看 Trace 的信息

![image-20200718163836919](https://i.loli.net/2020/07/18/j8ykWQdm9IeoJTD.png)

### Spring Boot 中使用 RabbitMQ

首先如果要使用 RabbitMQ 则需要在 POM 文件引入相关的依赖如下：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

属性配置如下：

```yaml
spring:
  rabbitmq:
    addresses: localhost
    username: guest
    password: guest
    virtual-host: /
```

需要使用 `@EnableRabbit` 注解启用，然后看一下自动配置的原理。从 `RabbitAutoConfiguration` 开始，其为容器中引入了 `rabbitConnectionFactory`  连接工厂，连接到 RabbitMQ Server, `RabbitTemplate` 用于发送接收 MQ 数据（类似 JDBC Template 等 Template）, `AmqpAdmin`  管理 AMPQ 创建删除 Exchange，Queue，Binding 等等。另外还为容器中引入了`RabbitAnnotationDrivenConfiguration` Config 类，这个配置类引入了一些配置信息。

`RabbitTemplate` 默认使用的是 `MessageConverter` 是 `SimpleMessageConverter`。

```java
private MessageConverter messageConverter = new SimpleMessageConverter();
```

在其 `createMessage` 方法中默认使用的是 `SerializationUtils.serialize(object);` 去序列化一个对象，转换成 byte 数组。

```log
content_type:	application/x-java-serialized-object
```

当然也可以不使用默认的 Message Convert 方法，其内部也提供了一些 Converter 方法类：

![](https://i.loli.net/2020/07/04/D35UVBGScXLdHoK.png)

另外在 AutoConfig 引入 `RabbitTemplate` 的时候会从容器中获取 Convert 类，作为默认的 Convert 类。

```java
@Bean
@ConditionalOnSingleCandidate(ConnectionFactory.class)
@ConditionalOnMissingBean(RabbitOperations.class)
public RabbitTemplate rabbitTemplate(RabbitProperties properties,
                                     ObjectProvider<MessageConverter> messageConverter,
                                     ObjectProvider<RabbitRetryTemplateCustomizer> retryTemplateCustomizers,
                                     ConnectionFactory connectionFactory) {
  PropertyMapper map = PropertyMapper.get();
  RabbitTemplate template = new RabbitTemplate(connectionFactory);
  messageConverter.ifUnique(template::setMessageConverter);
  //...
  return template;
}
```

所以可以通过为容器中导入一个 `MessageConverter` 的方式更改默认的 Convert 规则，例如：

```java
 @Bean
public MessageConverter messageConverter() {
  return new Jackson2JsonMessageConverter();
}
```

再次发送就会使用 JSON 的方式了：

```log
content_type:	application/json
```

####  Rabbit MQ 操作

发送消息：

```java
public void sendDemoUser() {
  DemoUser demoUser = new DemoUser();
  demoUser.setId(1);
  demoUser.setName("Hello World");
  demoUser.setPassword("DummyPassword");
  rabbitTemplate.convertAndSend("demo.direct", "usa.news", demoUser);
}
```

接收消息：

```java
public void receive(){
  Object o = rabbitTemplate.receiveAndConvert("usa.news");
  LogFactory.getLog(RabbitdemoApplicationTests.class.getSimpleName()).info(o);
}
```

使用 Listener 的方式接收消息：

```java
@RabbitListener(queues = "usa.news")
public void receive(DemoUser user) {
  log.info(user);
}

@RabbitListener(queues = "europe.news")
public void receiveMessage(Message message) {
  try {
    log.info(new ObjectMapper().readValue(message.getBody(), DemoUser.class));
  } catch (IOException e) {
    e.printStackTrace();
  }
  log.info(message.getBody());
  log.info(message.getMessageProperties());
}
```

管理 AMQP：

```java
public void admin() {
  Exchange exchange = new TopicExchange("java.topic", true, false);
  amqpAdmin.declareExchange(exchange);

  Queue queue = new Queue("xinyue", true, false, false);
  amqpAdmin.declareQueue(queue);

  Binding binding = new Binding("xinyue", Binding.DestinationType.QUEUE, "java.topic", "xinyue.#", new HashMap<>());
  amqpAdmin.declareBinding(binding);
}
```

### RabbitMQ 集群

#### RabbitMQ 自定义监控

RabbitMQ 提供了很多 resultFul 风格的 api 接口，可以通过这些接口带错对应的集群数据，可以通过这种方式定制监控系统。

| HTTP API URL                          | HTTP请求类型   | 接口含义                                                     |
| ------------------------------------- | -------------- | ------------------------------------------------------------ |
| /api/connections                      | GET            | 获取当前RabbitMQ集群下所有打开的连接                         |
| /api/nodes                            | GET            | 获取当前RabbitMQ集群下所有节点实例的状态信息                 |
| /api/vhosts/{vhost}/connections       | GET            | 获取某一个虚拟机主机下的所有打开的connection连接             |
| /api/connections/{name}/channels      | GET            | 获取某一个连接下所有的管道信息                               |
| /api/vhosts/{vhost}/channels          | GET            | 获取某一个虚拟机主机下的管道信息                             |
| /api/consumers/{vhost}                | GET            | 获取某一个虚拟机主机下的所有消费者信息                       |
| /api/exchanges/{vhost}                | GET            | 获取某一个虚拟机主机下面的所有交换器信息                     |
| /api/queues/{vhost}                   | GET            | 获取某一个虚拟机主机下的所有队列信息                         |
| /api/users                            | GET            | 获取集群中所有的用户信息                                     |
| /api/users/{name}                     | GET/PUT/DELETE | 获取/更新/删除指定用户信息                                   |
| /api/users/{user}/permissions         | GET            | 获取当前指定用户的所有权限信息                               |
| /api/permissions/{vhost}/{user}       | GET/PUT/DELETE | 获取/更新/删除指定虚拟主机下特定用户的权限                   |
| /api/exchanges/{vhost}/{name}/publish | POST           | 在指定的虚拟机主机和交换器上发布一个消息                     |
| /api/queues/{vhost}/{name}/get        | POST           | 测在指定虚拟机主机和队列名中获取消息，同时该动作会修改队列状态试 |
| /api/healthchecks/node/{node}         | GET            | 获取指定节点的健康检查状态                                   |
| /api/nodes/{node}/memory              | GET            | 获取指定节点的内存使用信息                                   |

可以通过 http://localhost:15672/api/index.html 页面看到相关的信息。可以通过程序获取这些监控数据，做自定义监控。

#### RabbitMQ 集群架构

RabbitMQ 集群架构有两种模式：

##### 主备模式

用来实现 RabbitMQ 的高可用集群，一般是在并发和数据不是特别多的时候使用，当主节点挂掉以后从备份节点中选择一个节点作为主节点对外提供服务。

![](https://i.loli.net/2020/07/18/mZW3TCKFztNJQUi.png)

消费者通过 HA Proxy 访问主节点，当主节点挂掉以后 HAProxy 会帮助我们把备份节点切换成主节点对外提供服务。

HA 的部分配置如下：

```shell
server s1 ip1:port check inter 5000 rise 2 fall 2 #主节点
server s2 ip2:port backup check inter 5000 rise 2 fall #2 备份节点
```

inter 代表每 5 秒钟对集群做一次健康检查， raise 2 代表 2 次正常则服务可用，2 次 失败代表服务失败服务不可用，backup 代表备份节点。

##### 远程模式

主要用来实现双活，简称 Shovel 模式，即让我们可以把消息复制到不同的数据中心，让两个地域集群互联。

![](https://i.loli.net/2020/07/18/4hkwLCIzNyVZO1Q.png)

##### 镜像模式

镜像队列也被称为 Mirror 队列，主要用来保证 MQ 消息的可靠性，其通过消息复制的方式保证消息 100% 不丢失，同时该集群模式也是企业中使用最多的模式。

![image-20200719000814483](https://i.loli.net/2020/07/19/Ng2AxHuKowhLbyO.png)

使用 HAProxy 做负载均衡，同时因为当前节点也可能挂掉，所以受用 KeepAlive 漂移到另一个 HAProxy 节点提供服务。任意一个 MQ 节点收到用户发过来的消息都会将消息复制给其他节点，任意一个节点挂掉，在其他节点都存在该消息，消息并不会随着一个节点挂掉而消失。

备注：HAProxy 是一款提供高可用性，负载均衡基于 TCP和HTTP应用代理软件，借助 HAProxy 可以快速可靠的提供基于 TCP/HTTP应用的代理解决方案。其适用于大型 web 站点，这些站点通常又需要会话保持或七层处理。HAProxy 可以支持数以万计的并发连接并且 HAProxy 的运行模式使得它可以很简单安全的整合进架构中，同时可以保护 Web 服务器不暴露在网络上。

KeepAlived 是一个高性能的服务器可用或热备解决方案，KeepAlived 主要来防止服务器单点故障的发生问题，可以通过其与 Nginx，HAProxy 等反向代理的负载均衡配合实现 Web 服务端的高可用，KeepAlived 以 VRRP 协议为实现基础，用 VRRP 协议来实现高可用性，VRRP 协议是用来实现路由器冗余的协议，VRRP 协议将两台或多台路由器设备虚拟成一个设备，对外提供一个虚拟路由 IP (一个或多个)。

##### 多活模式

多活模式主要用来实现异地数据复制，Shovel 模式其实也可以实现，但是其配置比较繁琐且受到版本的限制，如果做异地多活可以使用多活模式，即借助 Federation 插件来实现集群与集群之间或者节点与节点之间的消息复制。

![image-20200718235404692](https://i.loli.net/2020/07/18/etDumdvLgFEpNPI.png)

当用户发送一条 MQ 消息过来使用 LBS 做负载均衡，到了集群 1，集群 1 在接收到该消息后就会使用 Federation 插件把该条消息复制集群 2，集群一般情况都使用镜像队列的方式而不是使用集群与集群之间的复制，只需要将集群 1 的某一个节点的数据复制到集群 2 的某一个节点上，然后集群 2 的节点会使用镜像队列的方式自己去同步到所有的节点上。

### RabbitMQ 其他

#### 消息堆积

当生产消息的速度长时间远远大于消费速度的时候就会造成消息堆积。

消息堆积坑影响：导致新消息无法进入队列，就消息无法丢失，消息等待消费时间过长。

出现堆积情况的有以下几种情况：生产者突然大量发布消息，消费者消费失败，消费者出现性能瓶颈。

解决方案：排查消费者性能瓶颈，增加消费者的多线程处理，部署增加多个消费者。

#### 消息丢失

实际生产环境中可能出现一条消息因为一些原因丢失，导致消息没有消费成功，从而造成数据不一致等问题，主要场景分为：消息在生产者丢失，消息在 RabbitMQ 丢失，消息在消费丢失。

##### 在生产者丢失

采用 RabbitMQ 发送消息缺乏机制，当消息成功被 MQ 接收到的时候，会给生产者发送一个确认消息，表示接收成功，确认模式一般有以下三种：普通确认模式，批量确认模式，异步监听确认模式。Spring 中使用异步监听确认模式，边发消息边确认，不影响主线程任务执行。

##### 在 RabbitMQ 丢失

一般采用持久化交换机，队列，消息。确保 MQ 服务器重启时依然能够从磁盘回复对应的交换机，队列和消息。Spring 中默认开启了交换机，队列和消息的持久化。

##### 在消费丢失

设置手动回复 MQ 服务器，当消费者出现异常或者服务宕机的时候，MQ 服务器不会删除该消息，而是把消息重新绑定给队列的消费者，如果该队列只绑定了一个消费者，那么消息会一直保存在 MQ 服务器，直到消费者能够正常消费为止。MQ 重发消息主要场景有： 1 消费者未响应 ACK，主动关闭频道或连接；2 消费者未响应 ACK，消费者服务挂掉。

ACK 模式：

- `nack`/`reject` with `requeue=1`:  Message 会重新放到 Queue 中，用在消费端临时出现故障。
- `nack`/`reject` with `requeue=0`：如果配置了 DLX 会将 Message 放到 DLX 中，如果没有配置消息会被丢失
- `ack` 从 Queue 中删除消息。

reject 和 nack 的区别是，reject 只能针对于单条消息。

#### 有序消息消费

对于多个消费者产生的乱序消费情况，可以采取使用多个 Queue  每个 Queue 设置一个 Consumer，将需要顺序的处理的 Message 经过 hash 计算后放到同样的 Queue 中进行处理。

对于一个消费者采用多线程而出现乱序消费的情况，可以采用上面类似的方式，使用多个内存队列，多个线程分别从对应的内存队列中读取，想要顺序处理的 Message 放到相同的内存队列中。

#### 重复消息

如果消费消息的业务时幂等性的，即使出现重复消息也不会出现问题。对于不支持幂等性的操作，可以每次消费完之后将消息 ID 保存到数据库中，每次消费之前查看该消息，如果消息 ID 已经存在则表明已经消费过了，不再进行消费。可以使用 Redis，其有 setnx 命令可以使用。

### 