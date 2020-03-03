## Kafka 使用

### 发送

Kafka 的 Producer 发送消息时采用异步发送的方式，在消息的发送过程中，涉及到了两个线程， main 线程和 sender 线程，以及一个线程共享变量 `RecordAccumulator` main 线程将消息发送给 `RecordAccumulator` Sender 线程不断从  `RecordAccumulator` 中拉取消息发送到 Kafka Broker 上。如下图所示：

![发送流程](http://img.programya.com/Snipaste_2019-11-15_15-27-35.png)

```java
// 一个简单的 Producer
public class MyPartitionerProducer {
    static Logger logger = LoggerFactory.getLogger(MyPartitionerProducer.class);
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.29.129:9092");
        properties.put(ProducerConfig.ACKS_CONFIG, "all");
        properties.put(ProducerConfig.RETRIES_CONFIG, 1);
        properties.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        properties.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        properties.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");

        properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "com.xinyue.kfk.MyPartitioner");

        KafkaProducer<String, String> myProducer = new KafkaProducer<>(properties);
        for (int i = 0; i < 10; i++) {
            myProducer.send(new ProducerRecord<>("four", "key", "message::" + i), new Callback() {
                @Override
                public void onCompletion(RecordMetadata metadata, Exception exception) {
                    if (metadata != null) {
                        logger.info(metadata.toString());
                    }
                }
            });
        }
        myProducer.close();
    }
}
```

``` java
// 实现自己的 分区器
public class MyPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // key.hashCode() % cluster.partitionCountForTopic(topic)
        return 1;
    }
    @Override
    public void close() {}
    @Override
    public void configure(Map<String, ?> configs) {}
}

// 只有在发送配置中配置即可
properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "com.xinyue.kfk.MyPartitioner");
```

```java
// 可以通过 FutureTask 实现同步控制
for (int i = 0; i < 10; i++) {
    Future<RecordMetadata> future = myProducer.send(new ProducerRecord<>("four", "message::" + i));
    try {
        RecordMetadata recordMetadata = future.get();
        if (recordMetadata != null) {
            logger.info(recordMetadata.toString());
        }
    } catch (InterruptedException | ExecutionException e) {
        logger.error("Meet error", e);
    }
}
```



### 接收

```java
// 一个简单 消费者
public class MyConsumer {
    static Logger logger = LoggerFactory.getLogger(MyConsumer.class);
    public static void main(String[] args) {
        Properties properties = new Properties();
        // 连接集群
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.29.129:9092");
        // 自动提交
//        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        // 开启手动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // 自动提交延迟
        properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        // 反序列化设置
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        // 消费者组
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "xinyue-demo");
        // 从最早的 offset 开始读
//        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        KafkaConsumer<String, String> myConsumer = new KafkaConsumer<>(properties);
        myConsumer.subscribe(Collections.singletonList("four"));
        while (true) {
            ConsumerRecords<String, String> messages = myConsumer.poll(Duration.ofSeconds(1));
            if (messages != null && !messages.isEmpty()) {
                for (ConsumerRecord<String, String> message : messages) {
                    logger.info(message.toString());
                }
                // 同步提交，失败会重试
//                myConsumer.commitSync();
                // 异步提交
                myConsumer.commitAsync((offsets, exception) -> {
                    if (offsets != null) {
                        logger.info(offsets.toString());
                    }
                    if (exception != null) {
                        logger.error("Meet Error", exception);
                    }
                });
            }
        }

//        myConsumer.close();
    }
}
```

 在 Kafka 0.9 之后默认 offset 存储在 kafka 一个内置的 topic 中，除此之外kafka 可以选择自定义存储 offset。其实 offset 的维护是非常的繁琐，需要考虑到消费者的 Rebalance, 消费者发生 rebalace 之后，每个消费者消费的分区就会发生变化，因此消费者要首先获取到自己被重新分配的分区，并且定义到每个分区最近提交的 offset 的位置继续消费。要实现自定义存储 offset 需要使用 `ConsumerRebalanceListener` 如下所示：

```java
// 自定义 rebalance listener
myConsumer.subscribe(Collections.singletonList("four"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        commitOffset(currentOffset);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        currentOffset.clear();
        for (TopicPartition partition : partitions) {
            myConsumer.seek(partition, getOffset(partition)); //定位到最近提交的 offset 位置继续消费
        }
    }
});

// 存储到其他存储系统
// 自定义获取某分区的最新 offset
private static long getOffset(TopicPartition partition) {
    return 0;
}
// 自定义提交该消费者所有分区的 offset
private static void commitOffset(Map<TopicPartition, Long> currentOffset) {}
```



### 自定义拦截器

Producer 拦截器 interceptor 主要用于实现 client 端的定制化控制逻辑，对于 producer 而言， interceptor 使得用户在发送消息之前以及 producer 回调逻辑前有机会对消息进行一些定制化的需求，比如修改消息。同时 producer 允许用户指定多个 interceptor 按序作用于同一条消息从而形成一条拦截链。接口是 `org.apache.kafka.clients.producer.ProducerInterceptor` 主要定义了以下四个方法：

1. `configure`: 获取配置信息和初始化数据时调用。
2. `onSend` ：该方法封装在 KafkaProducer.send 方法中，即其运行在用户主线程中。Producer确保消息被序列化即计算分区之前调用此方法。用户可以在该方法中对消息做任何操作，但最好不要修改消息所属 topic 和分区，避免影响目标分区计算。
3. `onAcknowledgement`： 该方法从 ReacordAccumulator 成功发送到 Broker 之后，或者在发送过程中失败时调用。并且通常都是在 Producer 回调逻辑前触发。该方法运行在 producer 的 IO 线程中，因此不要在该方法中放入重逻辑，否则可能会导致 producer 发送效率变低。
4. `close`： 关闭 interceptor，主要用于一些资源清理工作。

interceptor 可能被运行于多个线程中，因此在具体实现时，需要实现者确保线程安全，此外如果指定了多个 interceptor，则 producer 会按照执行顺序调用它们，并仅仅捕获每个 interceptor 中可能抛出的异常到错误日志中，不会向上传递。

 ```java
// 自定义时间拦截器，在每条消息之前加上时间戳
public class TimeInterceptor implements ProducerInterceptor<String, String> {
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        String value = System.currentTimeMillis() + record.value();
        return new ProducerRecord<>(record.topic(), record.partition(), record.key(), value);
    }
    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {}
    @Override
    public void close() { }
    @Override
    public void configure(Map<String, ?> configs) {}
}

// 自定义计数拦截器，统计成功和失败的条数。
public class CounterInterceptor implements ProducerInterceptor<String, String> {
    static Logger logger = LoggerFactory.getLogger(CounterInterceptor.class);
    private int successCount = 0;
    private int errorCount = 0;
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        return record;
    }
    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (metadata != null) {
            successCount += 1;
        } else {
            errorCount += 1;
        }
    }
    @Override
    public void close() {
        logger.info("success::" + successCount);
        logger.info("error::" + errorCount);
    }
    @Override
    public void configure(Map<String, ?> configs) {}
}

// 使用的时候只需要在配置中添加即可
properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, Arrays.asList("com.xinyue.kfk.TimeInterceptor",
                "com.xinyue.kfk.CounterInterceptor"));

 ```

### 安装 Kafka Eagle 监控

Kafka Eagle 界面如下图所示：

![Eagle 界面](http://img.programya.com/Snipaste_2019-11-16_00-32-15.png)

修改 `kafka.server.start.sh` 文件：

```shell
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
	export KAFKA_HEAP_OPTS="-server -Xms2G -Xmx2G -XX:PermSize=128m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=5 -XX:InitiatingHeapOccupancyPercent=70"
	export JMX_PORT="9999"
	#export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi
```

设置环境变量

```shell
export KE_HOME=/usr/local/eagle
export PATH=.:$JAVA_HOME/bin:$PATH:$KAFKA_HOME/bin:$KE_HOME/bin
```

修改 `system-config.properties` 配置文件

```properties
kafka.eagle.zk.cluster.alias=cluster1
cluster1.zk.list=192.168.29.129:2181,192.168.29.130:2181,192.168.29.131:2181

cluster1.kafka.eagle.offset.storage=kafka

kafka.eagle.metrics.charts=true
kafka.eagle.sql.fix.error=false

kafka.eagle.driver=com.mysql.jdbc.Driver
kafka.eagle.url=jdbc:mysql://hadoop102:3306/ke?useUnicode=true&ch
aracterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
kafka.eagle.username=root
kafka.eagle.password=000000
```

通过 `./ke.sh start` 命令启动，如下图所示：

![启动成功标志](http://img.programya.com/Snipaste_2019-11-16_00-38-07.png)

登录即可查看相应的状态信息