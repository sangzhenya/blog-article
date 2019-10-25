## ActiveMQ 整合

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



