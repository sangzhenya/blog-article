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
9. Virtual Host 虚拟主机，表示一批交换器、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个 vhost 本质上就是一个mini 的 RabbitMQ 的服务器，用于自己的队列，交换器，绑定和权限机制。vhost 是 AMQP 概念的基础，必须在连接的时候指定。RabbitMQ 默认 vhost 是 /。
10. . Broker 表示消息队列服务器实体。

结构图如下：

![结构图](https://i.loli.net/2020/07/02/7Rpt26EzYDixM3j.png)

RabbitMQ 启动后默认使用两个端口，一个 15672 作为 Web 管理端口，一个是 5672 作为通讯端口。

### Rabbit MQ 运行机制

首先看一下 AMQP 中的消息路由，其和 Java 开发者熟悉的 JMS 存在一些差别，AMQP 中增加了 Exchange 和 Binding 的校色。生产中把消息发布到 Exchange上，消息最终到达队列并被消费者接收，而 Binding 决定交换器的消息应该发送到哪个队列。结构如下：

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

### Spring Boot 中使用 RabbitMQ

首先如果要使用 RabbitMQ 则需要在 POM 文件引入相关的依赖如下：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
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



