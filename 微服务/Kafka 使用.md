## Kafka 使用

### 发送

Kafka 的 Producer 发送消息时采用异步发送的方式，在消息的发送过程中，涉及到了两个线程， main 线程和 sender 线程，以及一个线程共享变量 `RecordAccumulator` main 线程将消息发送给 `RecordAccumulator` Sender 线程不断从  `RecordAccumulator` 中拉取消息发送到 Kafka Broker 上。如下图所示：

![发送流程](http://img.sangzhenya.com/Snipaste_2019-11-15_15-27-35.png)

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

