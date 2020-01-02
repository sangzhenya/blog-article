## Zookeeper 使用

Java 连接使用 Zookeeper 的样例

pom 依赖如下：

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.12.1</version>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.5.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.5.6</version>
</dependency>
```

简单的增加节点/获取节点/判断节点是否存在的操作。

```java
public class MyZookeeper {
    private static Logger logger = LoggerFactory.getLogger(MyZookeeper.class);
    private static final String CONNECT_STRING = "192.168.29.128:2181,192.168.29.129:2181,192.168.29.130:2181";
    private static final int SESSION_TIMEOUT = 2000;
    private static ZooKeeper zooKeeper;

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
        // 建立连接
        zooKeeper = new ZooKeeper(CONNECT_STRING, SESSION_TIMEOUT, watchedEvent -> {
            // watch 事件
            logger.info(watchedEvent.toString());
            try {
                List<String> children = zooKeeper.getChildren("/", true);
                for (String child : children) {
                    logger.info(child);
                }
            } catch (KeeperException | InterruptedException e) {
                e.printStackTrace();
            }
        });
        // 新增节点
        String path = zooKeeper.create("/node5", "node 5".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        logger.info(path);
        // 获取遍历节点
        List<String> children = zooKeeper.getChildren("/", true);
        for (String child : children) {
            logger.info(child);
        }
        // 判断节点是否存在
        Stat exists = zooKeeper.exists("/node1", false);
        logger.info(exists.toString());
        System.in.read();
    }
}
```

一个监控服务动态上下线的例子：

模拟 Server：

```java
public class ZooServer {
    private static Logger logger = LoggerFactory.getLogger(ZooServer.class);
    private static final String CONNECT_STRING = "192.168.29.128:2181,192.168.29.129:2181,192.168.29.130:2181";
    private static final int SESSION_TIMEOUT = 2000;
    private static ZooKeeper zooKeeper;

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
        zooKeeper = new ZooKeeper(CONNECT_STRING, SESSION_TIMEOUT, watchedEvent -> {});
        String server = String.valueOf(new Random().nextInt(10));
        String path = zooKeeper.create("/servers/server", server.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        logger.info("path::" + path);
        logger.info("server::" + server);
        System.in.read();
    }
}
```

模拟 Client：

```java
public class ZooClient {
    private static Logger logger = LoggerFactory.getLogger(ZooClient.class);
    private static final String CONNECT_STRING = "192.168.29.128:2181,192.168.29.129:2181,192.168.29.130:2181";
    private static final int SESSION_TIMEOUT = 2000;
    private static ZooKeeper zooKeeper;

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
        zooKeeper = new ZooKeeper(CONNECT_STRING, SESSION_TIMEOUT, watchedEvent -> {
            logger.info(watchedEvent.toString());
            try {
                List<String> children = zooKeeper.getChildren("/servers", true);
                for (String child : children) {
                    logger.info(child);
                    byte[] data = zooKeeper.getData("/servers/" + child, false, null);
                    logger.info(Arrays.toString(data));
                }
            } catch (KeeperException | InterruptedException e) {
                e.printStackTrace();
            }
        });
        List<String> children = zooKeeper.getChildren("/servers", true);
        for (String child : children) {
            logger.info(child);
        }
        System.in.read();
    }
}
```

