## ActiveMQ 高级

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