## Active MQ

面向消息的中间件 是指利用高效可靠的消息机制进行平台的数据交流，并给予数据通信进行分布式系统的集成。通过提供消息传递和消息排列模型在分布式环境下提供应用解耦、弹性伸缩、冗余存储、流量削风、异步通信、数据同步等功能。简单来说就是解耦，削峰，异步。通用的流程如下：

生产者将消息发送到消息服务器，消息服务器将消息存放在若干队列和主题中，在合适的时候，消息服务器会将消息转发给接受者。在这个过程中，发送和接收是异步的，即发送无需等待，而且发送者和接收者的生命周期也没有必然联系。在发布等于模式下，也可以处完成一对多的通讯。整个过程都是异步的，生产者和消费者不必了解对方，不必同时在线，只需要确认消息即可，。

### 安装查看

可以从官网 [ActiveMQ](https://activemq.apache.org/) 下, 解压缩后进入到 bin 目录下，运行 `activemq.bat start` 运行 ActiveMQ，默认进程端口是 61616，默认控制台地址是  http://localhost:8161/index.html，打开控制台地址显示下图则表明已经安装成功了。

![控制台](http://img.sangzhenya.com/Snipaste_2019-10-22_20-41-08.png)

使用默认的用户名和密码 `admin/admin` 进入控制台，显示如下：

![控制台](http://img.sangzhenya.com/Snipaste_2019-10-22_20-43-29.png)

### Java 编码实现

通用开发步骤

![JMS](http://img.sangzhenya.com/Snipaste_2019-10-22_21-20-48.png)



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

![Queue](http://img.sangzhenya.com/Snipaste_2019-10-22_21-54-15.png)

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

![Topic](http://img.sangzhenya.com/Snipaste_2019-10-22_21-55-45.png)

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



### JMS 开发

Java 消息服务， 是 JavaEE 中的一个技术。指的是两个应用程序直接进行异步通讯的 API，它为标准消息协议和消息服务提供了一组通用的接口，包括创建，发送，读取消息等，用于支持 Java 应用程序的开发。在 JavaEE 中当两个应用程序使用 JMS 进行通讯时，它们直接并不是直接相连的，而是通过一个共同的消息收发服务组件关联起来以达到解耦，异步，削峰的效果。

主要有以下四个内容：

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









