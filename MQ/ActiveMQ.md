---
title: "Active MQ"
tags: ["Active MQ"]
categories: ["MQ"]
date: "2019-06-01T09:00:00+08:00"
---

面向消息的中间件 是指利用高效可靠的消息机制进行平台的数据交流，并给予数据通信进行分布式系统的集成。通过提供消息传递和消息排列模型在分布式环境下提供应用解耦、弹性伸缩、冗余存储、流量削风、异步通信、数据同步等功能。简单来说就是解耦，削峰，异步。通用的流程如下：

生产者将消息发送到消息服务器，消息服务器将消息存放在若干队列和主题中，在合适的时候，消息服务器会将消息转发给接受者。在这个过程中，发送和接收是异步的，即发送无需等待，而且发送者和接收者的生命周期也没有必然联系。在发布等于模式下，也可以处完成一对多的通讯。整个过程都是异步的，生产者和消费者不必了解对方，不必同时在线，只需要确认消息即可，。

### 安装查看

可以从官网 [ActiveMQ](https://activemq.apache.org/) 下, 解压缩后进入到 bin 目录下，运行 `activemq.bat start` 运行 ActiveMQ，默认进程端口是 61616，默认控制台地址是  http://localhost:8161/index.html，打开控制台地址显示下图则表明已经安装成功了。

![控制台](http://img.programya.com/Snipaste_2019-10-22_20-41-08.png)

使用默认的用户名和密码 `admin/admin` 进入控制台，显示如下：

![控制台](http://img.programya.com/Snipaste_2019-10-22_20-43-29.png)

### Java 编码实现

通用开发步骤

![JMS](http://img.programya.com/Snipaste_2019-10-22_21-20-48.png)



1. 创建一个 connection factory
2. 通过 connect factory 来创建 JMS connection
3. 启动 JMS connection
4. 通过 connection  创建 JMS Session
5. 创建 JMS destination
6. 创建 JMS producer 或者创建 JMS Message 并设置 destination
7. 创建 JMS consumer 或者注册一个 JMS message listener
8. 发送或者接收 JMS Message
9. 关闭所有 JMS 资源



#### Queue

![Queue](http://img.programya.com/Snipaste_2019-10-22_21-54-15.png)

每个消息只能有一个消费者，一对一的关系，消息生产者和消费者之间上没有相关性，无论消费者在生产者发送消息的是否在线，消费者都可以提取消息。消息被消费后不会再存储，所以消费者不会消费已经被消费的消息。

##### Producer

```java
public class QProducer {
    private static final String URL = "tcp://localhost:61616";
    private static final String QUEUE_NAME = "DEMO.QUEUE.TEST001";
    public static void main(String[] args) throws JMSException, IOException {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
        Connection connection = connectionFactory.createConnection();
        connection.start();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Destination destination = session.createQueue(QUEUE_NAME);
        MessageProducer producer = session.createProducer(destination);
        for (int i = 0; i < 3; i++) {
            Message textMessage = session.createTextMessage("Hello " + i);
            producer.send(textMessage);
        }
        producer.close();
        session.close();
        connection.close();
    }
}
```

##### Consumer

```java
public class QConsumer {
    private static final String URL = "tcp://localhost:61616";
    private static final String QUEUE_NAME = "DEMO.QUEUE.TEST001";
    public static void main(String[] args) throws JMSException, IOException {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
        Connection connection = connectionFactory.createConnection();
        connection.start();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Destination destination = session.createQueue(QUEUE_NAME);
        MessageConsumer consumer = session.createConsumer(destination);
        /*
        // 同步阻塞式
        while (true) {
            TextMessage textMessage = (TextMessage) consumer.receive(4000);
            if (null != textMessage) {
                System.out.println("Receive Message..." + textMessage);
            } else {
                break;
            }
        }*/
        // 异步非阻塞式
        consumer.setMessageListener(message -> System.out.println("Receive Message..." + message));
        System.in.read();
        consumer.close();
        session.close();
        connection.close();
    }
}
```



#### Topic

![Topic](http://img.programya.com/Snipaste_2019-10-22_21-55-45.png)

##### Producer

```java
public class TProducer {
    private static final String URL = "tcp://localhost:61616";
    private static final String TOPIC_NAME = "DEMO.QUEUE.TEST001";
    public static void main(String[] args) throws JMSException, IOException {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
        Connection connection = connectionFactory.createConnection();
        connection.start();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Destination destination = session.createTopic(TOPIC_NAME);
        MessageProducer producer = session.createProducer(destination);
        for (int i = 0; i < 3; i++) {
            Message textMessage = session.createTextMessage("Hello " + i);
            producer.send(textMessage);
        }
        producer.close();
        session.close();
        connection.close();
    }
}
```



##### Consumer

```java
public class TConsumer {
    private static final String URL = "tcp://localhost:61616";
    private static final String TOPIC_NAME = "DEMO.QUEUE.TEST001";
    public static void main(String[] args) throws JMSException, IOException {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
        Connection connection = connectionFactory.createConnection();
        connection.start();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Destination destination = session.createTopic(TOPIC_NAME);
        MessageConsumer consumer = session.createConsumer(destination);
        /*
        // 同步阻塞式
        while (true) {
            TextMessage textMessage = (TextMessage) consumer.receive(4000);
            if (null != textMessage) {
                System.out.println("Receive Message..." + textMessage);
            } else {
                break;
            }
        }*/
        // 异步非阻塞式
        consumer.setMessageListener(message -> System.out.println("Receive Message..." + message));
        System.in.read();
        consumer.close();
        session.close();
        connection.close();
    }
}
```



生产者将消息发布到 Topic 中，每个消息可以有多个消费者，属于 1对 N 的关系。生产者和消费者之间有时间上的相关性。订阅了某个主题的消费者只能消费自其订阅之后发布的消息。生产者生产时，topic 不保存消息其实无状态的。此外 JMS 规范中允许创建持久订阅，这在一定程度上放松了时间上的相关性要求。持久订阅允许消费者消费它处于未激活状态时发送的消息。

#### Queue 和 Topic 比较

| 比较项目   | Topic                                                        | Queue                                                        |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 工作模式   | 订阅发布模式，如果当前没有订阅者，消息将会被丢弃，如果有多个订阅者，那么这些订阅者都会收到消息 | 负载均衡模式，如果当前没有消费者，消息也不会丢弃，如果有多个消费者那么一条消息只会发送给其中一个消费者，并且要求消费者 ack 消息 |
| 有无状态   | 无状态                                                       | Queue 数据默认会在 mq 服务器上以文件形式保存，比如 ActiveMQ 一般保存在 ￥AMQ_HOME\data\kr-store\data 下面。也可以配置在 DB 中存储 |
| 传递完整性 | 如果没有订阅者，消息会被丢弃                                 | 消息不会丢弃                                                 |
| 处理效率   | 由于消息要按照订阅者的数量进行复制，所以处理性能会随着订阅者的增加而明显降低，并且还要结合不同消息协议自身的性能差异 | 由于一条消息只发送给一个消费者，所以消费者多不会导致性能下降，不过不同消息协议的具体性能有差异。 |



### JMS 

#### 概要

Java 消息服务， 是 JavaEE 中的一个技术。指的是两个应用程序直接进行异步通讯的 API，它为标准消息协议和消息服务提供了一组通用的接口，包括创建，发送，读取消息等，用于支持 Java 应用程序的开发。在 JavaEE 中当两个应用程序使用 JMS 进行通讯时，它们直接并不是直接相连的，而是通过一个共同的消息收发服务组件关联起来以达到解耦，异步，削峰的效果。

#### 重要组成

1. JMS Provider： 实现 JMS 接口和规范的消息中间件，即 MQ 服务器。

2. JMS Producer：消息生产者，创建和发送 JMS 消息的客户端应用

3. JMS Consumer：消息消费者，接收和处理 JMS 消息的客户端应用

4. JMS Message：

   1. 消息头：

      JMS Destination：目的地

      JMS Delivery Mode：持久与非持久

      JMS Expiration：过期时间，默认永不过期

      JMS Priority：优先级， 0-4 普通，5-9 加急，默认是 4 

      JMS Message ID： 永不重复的 ID

   2. 消息体

      TestMessage：普通字符串消息

      MapMessage：Map 类型消息，key 为 String 类型，值为 Java 基本类型

      BytesMessage：二进制消息，包含一个 byte[]

      StreamMessage：Java 数据流消息，用标准流操作来顺序填充和读取

      ObjectMessage：对象消息，包含一个可序列化的 Java  对象

   3. 消息属性

      以属性名和属性值的形式指定的键值对，是消息头的扩展，指定一些附加信息。例如 `message.setStringProperty("auth", "Key)`

#### JMS 持久性/事务/签收

##### 持久性

非持久性时（`DeliveryMode.NON_PERSISTENT`）当服务器宕机，消息不存在；持久性时（`DeliveryMode.PERSISTENT`），服务器宕机，消息仍然存在。

Queue 上的消息默认是持久性的。

Topic  的持久性设置如下：

**Producer**

```java
public class TProducerPersistent {
    private static final String URL = "tcp://localhost:61616";
    private static final String TOPIC_NAME = "DEMO.QUEUE.TEST.PERSISTENT001";
    public static void main(String[] args) throws JMSException, IOException {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
        Connection connection = connectionFactory.createConnection();
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Destination destination = session.createTopic(TOPIC_NAME);
        MessageProducer producer = session.createProducer(destination);
        producer.setDeliveryMode(DeliveryMode.PERSISTENT);
        connection.start();
        for (int i = 0; i < 3; i++) {
            Message textMessage = session.createTextMessage("Hello " + i);
            producer.send(textMessage);
        }
        producer.close();
        session.close();
        connection.close();
    }
}
```

**Consumer**

```java
public class TConsumerPersistent {
    private static final String URL = "tcp://localhost:61616";
    private static final String TOPIC_NAME = "DEMO.QUEUE.TEST.PERSISTENT001";
    public static void main(String[] args) throws JMSException, IOException {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
        Connection connection = connectionFactory.createConnection();
        connection.setClientID("PERSISTENT001");
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Topic destination = session.createTopic(TOPIC_NAME);
        TopicSubscriber topicSubscriber = session.createDurableSubscriber(destination, "TopicSubscriber");
        connection.start();
        Message message = topicSubscriber.receive();
        while (null != message) {
            System.out.println("Receive Message..." + message);
            message = topicSubscriber.receive(3000);
        }
        session.close();
        connection.close();
    }
}
```



##### 事务

在 Producer 中如果设置 事务为 true，那么发送 Message 的时候需要手动调用 commit。

```java
// 设置事务为 true
Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);

// 手动 commit
session.commit();
// 手动 rollback
session.rollback();
```

在 Consumer 中如果设置事务为 true，那么接收 Message 后也需要手动调用 commit。

```java
// 设置事务为 true
Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);

// 手动 commit
session.commit();
// 手动 rollback
session.rollback();
```



##### 签收

主要有三种：自动签收（`Session.AUTO_ACKNOWLEDGE`）、手动签收（`Session.CLIENT_ACKNOWLEDGE`）、运行重复消息（`Session.DUPS_OK_ACKNOWLEDGE`），事务（`Session.SESSION_TRANSACTED`）

```java
// 设置手动签收
Session session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE);

// 需要手动签收
message.acknowledge();
```

在事务性会话中，当一个事务被成功提交则消息被自动签收，如果事务回滚，则消息会被再次传送。

非事务性的会话中，消息何时被确认取决于创建会话的应答模式。



### ActiveMQ Broker

ActiveMQ Broker 作为独立的消息服务器构建 Java 应用，也支持在 VM 中通信基于嵌入式的 broke，集成在 java 应用中。本质上 Broker 就是实现了用代码的形式启动 ActiveMQ 将 MQ 嵌入到 Java 代码炸，以便随时启动，在用的时候再去启动这样能节省资源。

```java
public class EmbedBroker {
    public static void main(String[] args) throws Exception {
        BrokerService brokerService = new BrokerService();
        brokerService.setUseJmx(true);
        brokerService.addConnector("tcp://localhost:61616");
        brokerService.start();
    }
}
```



### ActiveMQ 与 Spring

可以将 ActiveMQ 相关的类交给 Spring 管理。

`pom.xml` 中的依赖如下：

```xml
<dependencies>
    <!-- https://mvnrepository.com/artifact/org.apache.activemq/activemq-all -->
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-all</artifactId>
        <version>5.15.10</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.apache.xbean/xbean-spring -->
    <dependency>
        <groupId>org.apache.xbean</groupId>
        <artifactId>xbean-spring</artifactId>
        <version>4.14</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-api -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>2.0.0-alpha1</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/ch.qos.logback/logback-classic -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.3.0-alpha4</version>
        <scope>test</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.10</version>
        <scope>provided</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.junit.jupiter/junit-jupiter-api -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.6.0-M1</version>
        <scope>test</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.10.0</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework/spring-jms -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jms</artifactId>
        <version>5.2.0.RELEASE</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework/spring-core -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.2.0.RELEASE</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.apache.activemq/activemq-pool -->
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-pool</artifactId>
        <version>5.15.10</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.2.0.RELEASE</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework/spring-aop -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>5.2.0.RELEASE</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.springframework/spring-orm -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-orm</artifactId>
        <version>5.2.0.RELEASE</version>
    </dependency>
</dependencies>
```

Spring 配置文件`applicationContext.xml` 如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.xinyue.activemq" />

    <bean id="jmsFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
        <property name="connectionFactory">
            <bean class="org.apache.activemq.ActiveMQConnectionFactory">
                <property name="brokerURL" value="tcp://localhost:61616" />
            </bean>
        </property>
        <property name="maxConnections" value="100"/>
    </bean>

    <bean id="destinationQueue" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg index="0" value="DEMO.QUEUE.SPRING.TEST" />
    </bean>

    <bean id="destinationTopic" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg index="0" value="DEMO.TOPIC.SPRING.TEST" />
    </bean>

    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="jmsFactory" />
        <property name="defaultDestination" ref="destinationQueue" />
        <property name="messageConverter">
            <bean class="org.springframework.jms.support.converter.SimpleMessageConverter" />
        </property>
    </bean>

    <bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="jmsFactory" />
        <property name="destination" ref="destinationQueue" />
        <property name="messageListener" ref="myMessageListener" />
    </bean>
</beans>
```

Producer 代码：

```java
@Service(value = "qProducer")
public class QProducer {
    @Autowired
    private JmsTemplate jmsTemplate;

    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
        QProducer producer = (QProducer) ctx.getBean("qProducer");
        producer.jmsTemplate.send(session -> session.createTextMessage("Demo Message..."));
        System.out.println("Finish");
    }
}
```



Consumer 代码：

```java
@Service(value = "qConsumer")
public class QConsumer {
    @Autowired
    private JmsTemplate jmsTemplate;

    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
        QConsumer consumer = (QConsumer) ctx.getBean("qConsumer");
        Message message = consumer.jmsTemplate.receive();
        System.out.println("Receive Message From Queue::" + message);
    }
}
```



Listener 代码（启动即会开始监听）

```java
@Service(value = "myMessageListener")
public class QMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        System.out.println("Receive Message From Queue::" + message);
    }
}
```



### ActiveMQ 与 SpringBoot

在 SpringBoot 中使用 ActiveMQ

`pom.xml` 中的依赖配置

```java
<dependencies>
        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>2.2.0.RELEASE</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.2.0.RELEASE</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <version>2.2.0.RELEASE</version>
            <scope>test</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-activemq -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
            <version>2.2.0.RELEASE</version>
        </dependency>
</dependencies>
```

`application.yml` 配置

```xml
server:
  port: 8081
spring:
  activemq:
    broker-url: tcp://localhost:61616
    user: admin
    password: admin
  jms:
    pub-sub-domain: false  # false:queue; true:topic

queuename: DEMO.MQ.BOOT.TEST
topicname: DEMO.MT.BOOT.TEST
```

Java 程序：

```java
// 主启动类
@SpringBootApplication
public class BootMQApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootMQApplication.class);
    }
}

// Config 类
@Configuration
@EnableJms
@EnableScheduling
public class MQConfig {
    @Value("${queuename}")
    private String queueName;
    @Value("${topicname}")
    private String topicName;

    @Bean
    public Queue queue() {
        return new ActiveMQQueue(queueName);
    }

    @Bean
    public Topic topic() {
        return new ActiveMQTopic(topicName);
    }
}

// 生产者类
@Component(value = "qProducer")
public class QProducer {
    private final JmsMessagingTemplate jmsMessagingTemplate;
    private final Queue queue;

    public QProducer(JmsMessagingTemplate jmsMessagingTemplate, Queue queue) {
        this.jmsMessagingTemplate = jmsMessagingTemplate;
        this.queue = queue;
    }

    public void produce() {
        jmsMessagingTemplate.convertAndSend(queue, "Demo Message");
    }

    @Scheduled(fixedDelay = 3000)
    public void produceTask() {
        jmsMessagingTemplate.convertAndSend(queue, "Demo Message Task");
        System.out.println("Send message task done");
    }
}

// 消费者类
@Component(value = "qConsumer")
public class QConsumer {
    private final JmsMessagingTemplate jmsMessagingTemplate;
    private final Queue queue;

    public QConsumer(JmsMessagingTemplate jmsMessagingTemplate, Queue queue) {
        this.jmsMessagingTemplate = jmsMessagingTemplate;
        this.queue = queue;
    }

    @JmsListener(destination = "${queuename}")
    public void consume(TextMessage textMessage) {
        System.out.println("Receive TextMessage::" + textMessage);
    }

    public void consume() {
        String message = jmsMessagingTemplate.receiveAndConvert(queue, String.class);
        System.out.println("Receive TextMessage::" + message);
    }
}

```

### ActiveMQ 传输协议

ActiveMQ 中支持的  client-broker 通讯协议有：TCP、NIO、UDP、SSL、HTTP(s)、VM 等。在 `activemq.xml` 中的 `transportConnectors` 中。

```xml
<transportConnectors>
    <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
    <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
</transportConnectors>
```

1. TCP 协议

   默认的 Broker 配置，监听 61616 端口。在网络传输数据前必须要序列化数据，消息通过 wire protocol 序列化成字节流。默认情况下 wire protocol 即 OpenWire，其为了促使网络上的效率和数据快速交互。连接样式是 `tcp://hostname:port?[key=value]` 。有以下优点：1 TCP 协议传输可靠性高，稳定性强。2 高效性，字节流方式传递效率高。3 有效性，可用性，应用广泛，支持任何平台。

2. NIO 协议

   NIO 协议和 TCP 协议类似但 NIO 更侧重于底层访问操作。运行开发人员对同一资源可有更多的 client 调用和服务端有更多的负载。以下两种情况更适合 NIO 协议的场景： 1 可能有大连 client 连接到 Broker 上，一般情况下，大量的 Client 去连接 Broker 是被操作系统线程所限制的，NIO 的实现比 TCP 需要更少的线程去执行，所以推荐 NIO；2 对于 Broker 可能有一个迟钝的网络传输，NIO 可以提供更好的性能。连接样式是 `nio://hostname:prot?[key=value]`

3. AMQP 协议

   Advanced Message Queuing Protocol 一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端与消息中间件传递消息并不受客户端中间件不同产品，不同开发预语言等条件限制。

4. stomp 协议

   流文本定向消息协议，是一种 面向消息中间设计的简单协议。

5. SSL 协议

6. mqtt 协议

7. ws 协议

也可以协议也可以设置为 auto，如下配置：

```xml
<transportConnectors>
    <transportConnector name="auto+nio" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
</transportConnectors>
```



### ActiveMQ 消息持久化

为了避免意外宕机以后丢失消息，需要在重启后可以恢复消息队列，一般会使用持久化机制。对于 ActiveMQ 而言持久化机制有 JDBC，AMQ，KahaDB 和 LevelDB，基本处理逻辑是一致的。

首先生产者将消息发送出去后，消息队列会将消息存储在本地数据文件或内存数据库或远程数据库中，之后再试图将消息发送给消费者，成功之后则将消息从存储中删除，失败则继续尝试发送。消息队列启动以后首先要检查指定存储位置，如果有未发送成功的消息则会把消息发送出去。

默认的持久化方式是 kahaDB，主要配置是是在 `activemq.xml` 中的如下配置：

```xml
<!--
            Configure message persistence for the broker. The default persistence
            mechanism is the KahaDB store (identified by the kahaDB tag).
            For more information, see:

            http://activemq.apache.org/persistence.html
-->
<persistenceAdapter>
    <kahaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>
```

#### KahaDB

KahaDB 是默认默认的持久化方式，适用于任何场景，提供性能和恢复能力。消息存储使用一个事务日志和仅仅用一个索引文件来存储它所有的地址。其是一个针对于消息持久化的解决方案，对典型的消息使用模式进行了优化，数据被追加到 datalogs 中，当不需要 Log 文件中的数据时 log 文件也会被丢弃。

KahaDB 在消息保存目录中只有 4类文件和一个 lock，如下所示：

```
db.data
db.redo
db.free
db-1.log
lock
```

1. db-<number>.log 存储消息到预定义大小的数据记录文件中，当数据文件已满时，一个新的文件会随之创建，number 数值递增。当不再有引用到数据文件中任何消息时，文件会被删除或者归档。
2. db.data 包含持久化的 B 树索引，索引了消息数据记录中的消息。使用索引指向 db-<number>.log 里面存储的消息。
3. db.free 当前 db.data 文件里哪些页面是空闲的，文件具体内容是所有空闲页的 ID。
4. db.redo 用于消息回复，在强制退出后会使用，用于回复 B 树索引。
5. lock 文件锁，表示当前获得 KahaDB 读写权限的 broker。

#### JDBC

主要有三张表 `activemq_msgs`, `activemq_acks`, `activemq_lock`。

1. `activemq_msgs` 消息表

   id：自增主键

   container: 消息的 destination

   msgid_prod: 消息生产者的主键

   msg_seq: 发送消息的顺序，msggid_prod + msg_seq 可以组成 JMS 的 MessageID

   expiration: 消息过期时间

   msg：消息本地的 Java 序列化对象的二进制数据

   priority: 优先级  0 - 9

2. `activemq_acks` 订阅表，如果对于持久化的 Topic，订阅者和夫妻的订阅关系保存在这个表中。

   container: 消息的 destination

   sub_dest: 如果使用 Static 集群，此记录集群其他系统信息

   client_id： 每个订阅者都必须有一个唯一的 client id

   sub_name: 订阅者名称

   selector: 选择器，可以选择只消费满足条件的消息，条件可以用自定义属性实现，支持多属性的 and 和 or 操作

   last_acked_id: 记录消费过消息的 ID

3. `activemq_lock`‘ 集群环境中才有作用，只有一个 Broker 可以获得消息，称为 Master Broker，其他的只能作为备份等待 Master Broker 不可用，才可能成为下一个 Broker，该表记录当前的 Master Broker

`activemq.xml` 中配置如下，并且需要将 mysql-connect-java 的包放到 lib 目录下：

```xml
<persistenceAdapter>
    <jdbcPersistenceAdapter dataSource="#mysql-ds" />
</persistenceAdapter>

<bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/activemq?relaxAutoCommit=true&amp;serverTimezone=UTC"/>
    <property name="username" value="xinyue"/>
    <property name="password" value="xinyue"/>
    <property name="poolPreparedStatements" value="true"/>
</bean>
```

如果使用 JDBC with journal则使用如下的 persistence 配置。使用 ActiveMQ Journal 的告诉缓存写入技术，可以极大的提高性能，在消费者速度能够及时跟上生产者消息生产速度时，journal 文件能够大大减少写入到 DB 中消息。

```xml
<persistenceFactory>
    <journalPersistenceAdapterFactory
        journalLogFiles="4"
        journalLogFileSize="32768"
        useJournal="true"
        useQuickJournal="true"
        dataSource="#mysql-ds"
        dataDirectory="activemq-data">				  
    </journalPersistenceAdapterFactory>
</persistenceFactory>
```



### ActiveMQ 多节点集群

### ActiveMQ 异步、延时和定时投递

#### 异步

ActiveMQ 支持同步和异步两种发送模式将消息发送到 Broker，模式的选择对发送延时有巨大的影响。Producer 能达到怎样的产出率主要受发送时延的影响，使用异步发送可以显著提高发送性能。ActiveMQ 默认使用异步发送模式：除非明确指定使用同步发送的方式或者在未使用事务的前提下发送持久化的消息，这两种情况都是同步发送的。

如果在未使用事务的情况下发送持久化的消息，每次发送都是同步发送且会阻塞 Producer 直到 Broker 返回一个确认，表示消息已经被安全的持久化到磁盘。确认机制提供了消息安全的保障，但同时会阻塞客户端带来了很大的延迟。很多高性能的应用允许在失败的情况下有少量的数据丢失，如果你的应用满足这个特点，则可以使用异步发送来提高生产率，即使是发送持久化消息。

可以使用以下三种方式开启：

```java
// 连接参数中配置
"tcp://localhost:61616?jms.useAsyncSend=true"
// ConnectionFactory 配置参数
((ActiveMQConnectionFactory)connectionFactory).setUseAsyncSend(true);
// Connect 配置参数
((ActiveMQConnection)connection).setUseAsyncSend(true);
```

为了确保异步发送成功则需要接收回调，同步和异步的一个区别就是 同步 send 完成表示一定发送成功，而异步则需要接收回调由客户端判断是否发送成功。

```java
ActiveMQMessageProducer producer = (ActiveMQMessageProducer) session.createProducer(destination);
producer.send(textMessage, new AsyncCallback() {
    @Override
    public void onSuccess() {
        try {
            System.out.println(textMessage.getJMSMessageID() + "\tSend Success");
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onException(JMSException exception) {
        try {
            System.out.println(textMessage.getJMSMessageID() + "\tSend Faile");
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
});
```

#### 延时 和 定时

`activemq.xml` 中的 broker 的配置需要开启支持：

```xml
<broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" 
        dataDirectory="${activemq.data}" schedulerSupport="true">
    <!-- ... -->
</broker>
```

可以通过设置属性设置延时和定时

```java
// 延时 3 秒发送
textMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, 3000);
// 3 秒发送一次
textMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD, 3000);
// 发送三次
textMessage.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT, 3);
```

### ActiveMQ 消息重发、死信队列、防止重复调用

以下三种情况均会引起消息重发，间隔 1秒，重试 6 次

1. Client 使用事务，且在 Session 中调用 了 rollback
2. Client 使用事务，且在 调用 commit 之前关闭或者没有 commit
3. Client 在 Session.CLIENT_ACKNOWLEDGE 的模式下，在 session 中调用了 recover

如果一个消息被  redelivedred 超过最大重发次数（默认 6 次）之后，消费端会发送给 MQ 一个 `poison ack` 表示这个消息有毒，通知 Broker 不再发送该消息，此时 Broker 会把这个消息放到 DLQ 死信队列中。

可以在 Java 代码中手动修改默认参数：

```java
ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);
RedeliveryPolicy redeliveryPolicy = new RedeliveryPolicy();
redeliveryPolicy.setMaximumRedeliveries(3);
connectionFactory.setRedeliveryPolicy(redeliveryPolicy);
```

默认的死信队列叫做 `ActiveMQ.DLQ` 可以通过 deadLetterQueue 属性设定

```xml
<deadLetterStrategy>
	<sharedDeadLetterStrategy deadLetterQueue="DLQ-QUEUE" />
</deadLetterStrategy>
```

也可以单独指定死信队列：

使用 useQueueForQueueMessages 属性表示是否将 Topic 的 DeadLetter 保存到 Queue 中。默认 true

使用 processExpired 属性表示否是将过期消息放入私信队列。默认 true

使用 processNonPersistent 属性表示是否将非持久消息放入死信队列。默认 false

```xml
<policyEntry queue="DEMO.QUEUE.TEST">
	<deadLetterStrategy>
        <individualDeadLetterStrategy queuePrefix="DLQ." useQueueForQueueMessages="false" />
    </deadLetterStrategy>
</policyEntry>
```

对于重复消息的问题，可以使用一个第三方的服务器例如 Redis，给消息分配全局 id，如果消费过该消息，将 <id, message> 存储下来，消费开始前先去redis 中查询，从而避免重复消费。



