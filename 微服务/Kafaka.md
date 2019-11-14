## Kafka

kafka 是一个分布式的记于订阅发布模式的消息队列，主要应用于大数据实时处理领域。

**消息队列好处**：解耦，可恢复性，缓冲，灵活性与峰值处理能力，异步通讯。

**消息队列有两种模式**：点对点模式即一对一，发布订阅模式即一对多。

### Kafka 基础架构

如下图所示：

![kafka 基础架构]( http://img.sangzhenya.com/Snipaste_2019-11-14_20-53-45.png )

上图中主要组成有以下几部分：

1. Producer： 消息生产者
2. Consumer： 消息消费者
3. Customer Group：消费者组，由多个 Customer 组成。消费者组内每个消费者负责消费的区域不同分区的数据，一个分区只能由一个组内消费者，消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个消费者。
4. Broker：一台 kafka 服务器就是一个 broker，一个集群有多个 broker 组成，一个集群由多个 broker 组成。一个 broker 可以容纳多个 Topic。
5. Topic：消息队列，生产者和消费者面向都是一个 Topic。
6. Partition：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker 上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列。
7. Replica：副本，为保证集群中每个节点发生故障时，该节点上的 partition 数据不丢失且 kafka 仍然能够继续工作，其提供了一个副本机制，一个 topic 的每个分区都有若干个副本，一个 leader 和 多个 follower。
8. Leader：每个分区多个副本的主，生产者发送数据的对鞋，以及消费者消费数据的对象都是 Leader。
9. Follower：每个分区多个副本中的从，实时从 Leader 中同步数据，保持和 Leader 数据的同步。Leader 发生故障的时候 某个 Follower 会成为新的 Leader。



### Kafka 配置即启动

`server.properties` 中的几项配置：

```properties
# broker 的 id，必须是一个唯一的整数
broker.id=0
# 默认端口号 9092
#listeners=PLAINTEXT://:9092
num.network.threads=3
# 用于处理 IO 处理的线程数
num.io.threads=8
# send buffer
socket.send.buffer.bytes=102400
# receive buffer
socket.receive.buffer.bytes=102400
# 最大字节
socket.request.max.bytes=104857600
# 数据地址
log.dirs=/tmp/kafka-logs
# topic 在当前 broker 的分区个数
num.partitions=1
# 用来恢复和清理 data 下数据的线程数量
num.recovery.threads.per.data.dir=1
# segment 文件保留的最长时间，超时将被删除
log.retention.hours=168
# 配置连接 Zookeeper 的集群地址
zookeeper.connect=192.168.29.128:2181,192.168.29.129:2181,192.168.29.130:2181
```

在配置好之后使用 `/kafka-server-start.sh -daemon config/server.properties` 启动 `kafka` 集群。

常用的命令行命令如下：

```shell
bin/kafka-topics.sh --zookeeper 192.168.29.128:2181 --list
```



















