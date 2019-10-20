## Active MQ

面向消息的中间件 是指利用高效可靠的消息机制进行平台的数据交流，并给予数据通信进行分布式系统的集成。通过提供消息传递和消息排列模型在分布式环境下提供应用解耦、弹性伸缩、冗余存储、流量削风、异步通信、数据同步等功能。简单来说就是解耦，削峰，异步。通用的流程如下：

生产者将消息发送到消息服务器，消息服务器将消息存放在若干队列和主题中，在合适的时候，消息服务器会将消息转发给接受者。在这个过程中，发送和接收是异步的，即发送无需等待，而且发送者和接收者的生命周期也没有必然联系。在发布等于模式下，也可以处完成一对多的通讯。整个过程都是异步的，生产者和消费者不必了解对方，不必同时在线，只需要确认消息即可，。

### 安装查看

#### 控制台说明

### Java 编码实现

#### Queue

同步阻塞方式

异步非阻塞方式

#### Topic

#### Queue 和 Topic 比较

### JMS 开发

Java 消息服务

#### JMS 开发步骤

1. 创建一个 connection factory
2. 通过 connect factory 来创建 JMS connection
3. 启动 JMS connection
4. 通过 connection  创建 JMS Session
5. 创建 JMS destination
6. 创建 JMS producer 或者创建 JMS Message 并设置 destination
7. 创建 JMS consumer 或者注册一个 JMS message listener
8. 发送或者接收 JMS Message
9. 关闭所有 JMS 资源



