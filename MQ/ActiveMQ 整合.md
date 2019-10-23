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

### ActiveMQ 与 SpringBoot