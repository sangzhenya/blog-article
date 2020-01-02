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
# 默认端口号 9092; 设置每台机器各自的 ip 地址
listeners=PLAINTEXT://192.168.29.128:9092
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
# 每个 segment 最大的 size
log.segment.bytes=1073741824
```

在配置好之后使用 `./kafka-server-start.sh -daemon config/server.properties` 启动 `kafka` 集群。

常用的命令行命令如下：

```shell
# 列出所有的 topic
./kafka-topics.sh --zookeeper 192.168.29.129:2181 --list
# 创建 topic。 --topic 指定 topic 名称， --replication-factor 指定副本数， --partitions 定义分区数
# 副本数一定要小于集群中机器数量。
./kafka-topics.sh --zookeeper 192.168.29.129:2181 --create --replication-factor 1 --partitions 3 --topic first
# 删除 Topic
./kafka-topics.sh --zookeeper 192.168.29.129:2181 --delete --topic first
# 发送消息
./kafka-console-producer.sh --broker-list 192.168.29.129:9092 --topic first
# 消费消息  --from-beginning 从最开始读取
./kafka-console-consumer.sh --bootstrap-server 192.168.29.129:9092 --from-beginning --topic first
# 查看一个 topic 的详情 
./kafka-topics.sh --zookeeper 192.168.29.129:2181 --describe --topic first
# 修改分区数
./kafka-topics.sh --zookeeper 192.168.29.129:2181 --alter --topic first --partitions 6
```

查看 topic 状态

![查看 topic 状态]( http://img.sangzhenya.com/Snipaste_2019-11-15_11-08-54.png )



### Kafka 工作流程

![kafka 工作流程图]( http://img.sangzhenya.com/Snipaste_2019-11-15_11-46-17.png )

kafka 中消息时以 topic 进行分类的，生产者和消费者都是面向 topic 的。Topic 是逻辑上的概念，而 Partition 是屋里上的概念，每个 partition 对应于一个 log 文件，该 log 文件存储的就是 producer 生产的数据。Producer 生产的数据会被不断的追加到该 log 文件末端，且每条数据都有自己的 offset。消费者组中的每个消费者，都会实时记录自己的消费到了哪个 offset，以便出错恢复的时候能够继续从上次的位置继续消费。

![topic]( http://img.sangzhenya.com/Snipaste_2019-11-15_11-54-04.png )

由于生产者生产的消息会不断追加在到 log 文件末尾，为防止 log 文件过大导致数据定位效率低下，kafka 采用了分片和索引的机制，将每个 partition 分为多个 segment，每个 segment 对应两个文件：index 文件和 log 文件，这些文件位于一个文件夹内，该文件夹的命名规则为： topic 名称 + 分区序号。如下图所示：

![分区]( http://img.sangzhenya.com/Snipaste_2019-11-15_11-58-08.png )

index 和 log 文件的义当前 segment 的第一条消息的 offset 命名，如下图所示：

![](http://img.sangzhenya.com/Snipaste_2019-11-15_12-01-10.png)

index 文件存储了大量的索引信息， log 文件存储大量的数据，索引文件中元数据指向对应数据文件中的 message 的物理偏移地址。

### Kafka 生产者

Kafka 采用分区的原因主要是两个：首先方便集群管理扩展，每个 partition 可以通过调整以适应它所在的机器，而一个 topic 又可以有多个 partition 组成，因此整个集群可以适应任何大小的数据。其次可以提高并发，可以按照 partition 为单位读写。

对于 Java 编程来说，发送数据需要将数据包装成 ProducerRecord 对象。可以直接指定 partition 的值，如果没有指明 partition的值而是仅仅指定了可以，那么会将 key 的hash 值与 topic 的 partition 树进行取余得到 partition 的值。如果既没有 partition 又没有 key 的情况，第一次调用的时候会生成一个随机整数，之后每次都在这个整数上递增，这个值与 topic 可用的 partition 总数取余得到 partition的值，也就是 round-robin 算法。

为了保证 producer 发送的数据，能可靠的发送到指定的 topic，topic 的每个 partition 收到 producer 发送的数据后，都需要向 producer 发送 ack，如果 producer 收到 ack 则进行下一轮的发送，否则就重新发送数据。

![producer 发送数据]( http://img.sangzhenya.com/Snipaste_2019-11-15_13-27-47.png )

副本数据同步策略：一是半数以上完成同步就发送 ack，优点是延迟低，缺点是 选举新的 leader 时，容忍 n 台节点故障，需要 2n + 1 个副本。二是全部完成同步，才发送 ack，优点是容忍 n 台节点的故障，只需要 n+1 个副本，缺点是延迟高。kafka 采用的是方案二原因是 kafka 的每个分区都有大量 杜珊珊，第一种方案会造成大量的数据冗余，而网络延迟对 kafka 的影响相对小。

为了避免出现 Leader 收到数据，所有 follower 都开始同步数据，但只有一个 follower 因为某种故障没能及时同步的情况， Leader 维护了一个 动态的 in-sync replica set 即 ISR，也就是 和 leader 保持同步的 follower 集合。当 ISR 中的 follower 完成数据同步之后，就会发送 ack，如果 follower 长时间未向 leader 同步数据，那么该 follower 会从 ISR 集合中删除，该时间的阈值是由 `replica.lag.time.max.ms` 参数设置。 Leader 发生故障后，就会从 ISR 中选举新的 Leader。

对于一些不太重要的数据，对数据的可靠性的要求不是很高，能够容忍数据的少量丢失，所以没有必要等 ISR 中的 follower 全部接收成功，所以 kafka 提供了三种可靠性级别，用户根据对可靠性和延迟进行权衡。

acks 设置为 0 ，表示不需要 broker 的 ack，这一操作可以提供一个最低延迟，broker 接收到之后没有写入磁盘就已经返回，当 broker 故障的时候将会丢失数据。

acks 设置为 1，表示 producer 等待 broker 的ack，partition的 leader 接收成功之后返回 ack，如果 follower 同步成功之前 leader 故障，将会丢失数据。

acks 设置 -1，表示 producer 等待 broker 的ack， partition 的 leader 和 follow 全部成功之后才返回 ack，但是如果 follower 同步成功之后，broker 发送 ack 之前，leader 发生故障，那么会造成数据重复。

![数据重复的例子]( http://img.sangzhenya.com/Snipaste_2019-11-15_13-57-04.png )

故障的处理如下：

![故障处理]( http://img.sangzhenya.com/Snipaste_2019-11-15_13-58-15.png )

 LEO 是每个副本最大的 offset，HW 值当前消费者能见到的最大 offset， ISR 队列中最小的 LEO。

如果是 follower 故障，那么 follower 发生故障后就被从 ISR 中移除，等到 follower 恢复之后，follower 会读取本地磁盘记录的上一次的 HW，并将 Log 文件中高于 HW 的部分截掉，从 HW 开始想 Leader 进行同步，等到 follower 的 LEO 等于等于该 partition 的 HW ，即 follower 追上了 leader 之后，就可以重新假如 ISR 了。

若干是 leader 发生故障，那么会从 ISR 中选出一个新的 Leader，为表征多个副本之间的数据一致性，其余的 follower 会先讲各自的 log 文件高于 HW 的部分截掉然后，从新 leader 同步数据。

将服务器的 ack 级别设置为 -1 可以保证 producer 到 server 之间不会丢失数据，即 at least once 语义。相对的如果设置为 0 则表示生产者每条消息只会被发送一次，即 at most once 语义。

At Least Once 可以保证数据不丢失，但是不能保证数据不重复，相对的 At Most Once 可以保证数据不重复，但是不能保证数据不丢失。但是对于一些非常重要的信息，例如交易数据，下游系统消费者要求数据既不重复也不丢失，即 Exactly Once 语义，在 Kafka 中引入了一个叫做幂等性的特性，即 produce 不论向 Server 发送多少此重复数据， Server 端之后持久化一条。幂等性结合 At Least Once 语义，就构成了 Kafka 的 Exactly Once 语义。要启用幂等性，只需要在 Producer 参数中 `enable.idompotence` 设置为 true 即可。kafka 的幂等性的实现其实就是将原来下游需要做的去重放在了数据上游。开启幂等性后， Produce 在初始化的时候会分配一个 PID，发送同一 Partition 的消息会附带 Sequence Number，而 Broker 端会对 <PID, Partition, SeqNumber> 做缓存，当具有相同主键的消息提交时， Broker 仅仅会持久化一条。当时 PID 重启就会发生变化，同时不同的 Partition也就有不同的主键，所以 幂等性无法保证跨区跨会话的 Exactly One。

### Kafka 消费者

customer 采用 pull 模式从 broker 读取数据。因为 push 模式很难适应速率不同的消费者，因为 发送速率是由 broker 来决定的，push 的目标就是尽可能的以最快的速度传递消息，但是这样容易造成 consumer 来不及处理消息，典型表现就是拒绝服务以及网络拥塞。而 pull 模式则可以根据 consumer 的消费能力以适当的速率消费消息，不过 pull 模式的不足就是 kafka 没有数据的情况，消费者可能会陷入循环中，一直返回空数据，不过 kafka 的消费者在消费数据时会传入一个时长参数 timeout，如果当前没有数据可供消费， consumer 会等待一段时间在回复，即 timeout。

一个 customer group 中有多个 consumer，一个 topic 有多个 partition，所以会有 partition 分配的问题。kafka 有两种分配策略，一种是 RoundRobin （轮询） 把 topic 当做一个整体看待，将所有分区依次分配给 Customer，第二个是 Range 也是默认配置，是将单独的 topic 作为整体，将其中的分区分配给 Customer，对于一个 topic 而言，就是总数量处理 customer 数得到每个 customer 应该的消费的分区数 n，然后每 n 个分区分配给一个 Customer。对于同一个消费者组的中消费者，同一时刻只能有一个消费者消费。

由于 consumer 在消费过程中可能出现断电宕机等故障， consumer 恢复后，需要从故障前的位置继续消费，所以 consumer 需要实时记录自己消费到了哪个  offset，以便恢复后继续消费。kafka 从 0.9 版本开始 consumer 默认将 offset 保存在 kafka 一个内置的 topic 中，即 _consumer_offsets。 可以设置 ` consumer.properties` 中的 `exclude.internal.topics=false` 让 Customer 可以消费 internal 的 topic，进而可以获取 offset。

### Kafka 与 Zookeeper

kafka 的 producer 生产数据，要写入到 log 文件中，写的过程是一直追加到文件末端是顺序写，顺序写的速度远远大于随机写。另外在写的过程是零拷贝的，是不经过用户空间，而是直接由系统处理，如下图所示：

![零拷贝]( http://img.sangzhenya.com/Snipaste_2019-11-15_15-09-40.png )

kafka 集群中有一个 broker 会选举为 Controller，负责管理集群 broker 的上下线，所以 topic 的分区副本分配和 leader 选举工作，而 controller 的管理工作是依赖于 Zookeeper 的，如下面选择分区 leader 的过程：

![分区选取leader](http://img.sangzhenya.com/Snipaste_2019-11-15_15-05-21.png)

#### Kafka 事务

kafka 在 0.11 版本开始加入了对于事务的支持，保证了在 Exactly Once 语义基础 上的，生产和消费可以跨分区和会话的事务。

对于 producer ，为了实现跨分区会话的事务，需要引入一个全局唯一的 TransactionID 并将 Producer 的 PID 和 TransactionID 绑定，这样 Producer 重启之后可以通过正在进行的 TransactionID 获得原来的 PID，为了管理 Transaction， Kafka 引入了一个新的组件 Transaction Coordinator， Producer 就是通过和 Transaction Coordinator 交互获得 Transaction 对应的任务状态。Transaction Coordinator 还负责将事务所有写入 kafka 的一个内部 的Topic ，这样即使整个服务重启了，由于事务状态得到保存，进行中的事务可以得到恢复，从而继续进行。

对于 consumer，事务的包装相对较弱，无法保证  commit 信息被精确消费，因为 customer 可以通过 offset 访问任何信息，而且不同的 segment file 生命周期不同，同一事务的消息可能出现重启后被删除的情况。