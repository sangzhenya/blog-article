## Zookeeper

Zookeeper 从设计模式角度来看是一个机遇观察者模式的设计的分布式服务管理框架，负责存储和管理需要关心的数据，接受观察者注册，一旦观察的数据状态发生变化，Zookeeper 将负责将通知已经在 Zookeeper 上注册的那些观察者做出相应的反应。Zookeeper 结构图如下：

![Zookeeper结构图]( http://img.programya.com/Snipaste_2019-11-08_21-29-52.png )

 **Zookeeper 特点：**

1. 一个领导者和多个跟随着组成的集群。
2. 集群中只要有半数以上的节点存活， Zookeeper 就能正常服务。
3. 全局数据一致：每个 Server 保存一份相同的数据副本，客户端无论连接到哪个 Server，数据都是一致的。
4. 更新请求顺序进行，来自同一个客户端的更新请求按其发送顺序依次执行。
5. 数据更新具有原子性，一次数据要么成功，要么失败。
6. 具有实时性，在一定时间范围内， 客户端能读到最新数据。

**Zookeeper 数据结构：**

![Zookeeper 数据结构]( http://img.programya.com/Snipaste_2019-11-08_21-46-09.png )

Zookeeper 数据模型的结构与 Unix 文件系统类似，整体上可以看做是一棵树，每个节点乘坐一个 ZNode。每个 Znode 默认能够存储 1MB 的数据，每个 ZNode 都可以通过其路径唯一标识。

**应用场景**

Zookeeper 主要提供统一命名服务，统一配置管理，统一集群管理，服务器节点动态上下线，软负载均衡等等。

**安装**

首先安装 JDK，可以使用 OpenJDK

`sudo apt-get install openjdk-8-jdk`

设置环境变量路径如下：

```properties
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
PATH=.:$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME
export PATH
export CLASSPATH
```

下载 Zookeeper 并解压

```shell
axel -n 3 https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/current/apache-zookeeper-3.5.6-bin.tar.gz
tar -zxvf apache-zookeeper-3.5.6-bin.tar.gz
```

到 conf 目录中将 zoo.sample.conf 复制一份命名为 zoo.cfg。在 `/tmp` 中新建 zookeeper 文件夹，最后在 bin 目录中使用 `./zkServer.sh start` 启动 Zookeeper。使用 `jps` 或者 `./zkServer.sh status` 查看是否启动状态。

![Zookeeper 状态]( http://img.programya.com/Snipaste_2019-11-09_10-41-49.png )

Zookeeper 配置如下：

```properties
# 心跳帧间隔 2s
tickTime=2000
# 通讯正常后最大延迟心跳帧 5 * 2 = 10s
syncLimit=5
# 启动时最大延迟心跳帧 10 * 2 = 20s
initLimit=10
# 数据目录
dataDir=/tmp/zookeeper
# 端口
clientPort=2181
```

**选举机制**

Zookeeper 是集群中半数以上的集群存活则集群可用，所以适合安装在奇数台的服务器上。虽然 配置文件中并没有指定 Master 和 Slave，但是 Zookeeper 在工作的时候是有一个节点为 Leader 其他节点为 Follower，Leader 是通过内部的选举机制临时产生的。以五台服务器的集群为例：

首先服务器 1 启动，此时只有一台服务器启动，其发出去的报文没有任何响应，所以其选举状态一直是 Looking 状态。

然后服务器 2 启动，其余最开始启动的服务器1 进行通讯，互换自己的选举结果，由于二者均没有历史数据，所以 id 较大的服务器 2 胜出，但是由于没有达到超过半数的以上的服务器同意选举，所以服务器 1 和 2 均继续是 Looking 的状态。

然后服务器 3 启动，此时服务器 3 胜出，成为服务器 1,2,3 中的 Leader。

然后服务器 4 和 5 分别启动，但是此时服务器 3 已经是 Leader，所以 4 和 5 只能 follower 的角色。

**数据结构**

![数据结构]( http://img.programya.com/Snipaste_2019-11-09_11-07-21.png )

Zookeeper 有两种数据类型：持久和临时。

持久（Persistent）节点：客户端与服务器断开连接之后，创建的节点不删除。

临时（Ephemeral） 节点：客户端和服务器断开连接之后，创建的节点自己删除。

在创建 Znode 的时候可以设置一个顺序的标识，znode 名称会附加一个值，顺序号值是一个单调递增的计数器，由父节点维护。在分布式系统中可以被用于为所有事件进行全局排序，客户端就可以通过顺序号推断事件顺序。

上图中总共有四种类型：

1. 持久化目录节点：客户端断开后节点仍然存在。
2. 持久化顺序编号目录节点：客户端断开后，节点依旧存在，只是 Zookeeper 给节点名称进行顺序编号。
3. 临时目录节点：客户端断开后节点被删除。
4. 临时顺序编号目录节点：客户端断开连接后，节点被删除，只是 Zookeeper 给节点进行顺序编号。

**Zookeeper 集群搭建**

在 data 目录下新建 myid 为名的文件，写入 id 号，以三台服务器为例，三台分别写入 2,3,4 数字。

修改各个 Zookeeper 的 conf 文件添加对应的 ip 地址：

```properties
# server.id=ip:通讯端口:选举端口
server.2=192.168.29.128:2888:3888
server.3=192.168.29.129:2888:3888
server.4=192.168.29.130:2888:3888
```

配置完成后直接逐个启动服务即可，可以通过 `./zkServer status` 查看状态：

![查看集群状态]( http://img.programya.com/Snipaste_2019-11-09_11-42-37.png )

**基本客户端 Shell 操作**

```shell
# 帮助
help
# 查看当前 Znode 包含的内容
ls [-s] [-w] [-R] path
# 查看当前节点即更新次数等
ls2 path [watch]
# 创建 -s 包含序列号。-e 临时节点
create [-s] [-e] [-c] [-t ttl] path [data] [acl]
# 获取节点值
get [-s] [-w] path
# 设置节点值
set [-s] [-v version] path data
# 查看节点状态
stat [-w] path
# 删除节点
delete [-v version] path
# 递归删除节点
rmr path
```

**stat 结构体信息**

```
czxid: 创建节点的事务zxid
每次修改ZooKeeper状态都会收到一个zxid形式的时间戳，也就是ZooKeeper事务ID。
事务ID是ZooKeeper中所有修改总的次序。每个修改都有唯一的zxid，如果zxid1小于zxid2，那么zxid1在zxid2之前发生。

ctime: znode被创建的毫秒数(从1970年开始)
mzxid: znode最后更新的事务zxid
mtime: znode最后修改的毫秒数(从1970年开始)
pZxid: znode最后更新的子节点zxid
cversion: znode子节点变化号，znode子节点修改次数
dataversion: znode数据变化号
aclVersion: znode访问控制列表的变化号
ephemeralOwner: 如果是临时节点，这个是znode拥有者的session id。如果不是临时节点则是0。
dataLength: znode的数据长度
numChildren: znode子节点数量
```

如下图所示：

![stat 结构]( http://img.programya.com/Snipaste_2019-11-09_12-01-57.png )

**监听器原理**

在 main 线程中创建 zookeeper 客户端时会创建两个线程，一个负责网络通讯(connect)，一个负责监听(listener)。通过 connect 线程将注册监听的时间发送个 Zookeeper，在 Zookeeper 的注册监听器列表中将注册监听的事件添加到列表中。Zookeeper 监听到了 数据或者路径的变化后将会将消息发送给 listener 线程，调用 process 方法处理。

![watch 原理]( http://img.programya.com/Snipaste_2019-11-09_12-09-14.png )

**写数据的流程**

1. 首先客户端向 Zookeeper 的 Server 发送一个写请求。
2. 如果 Server 不是 Leader，那么会把这个请求进一步转发给 Leader ，因为每个 Zookeeper 中有一个 Leader 这个 Leader 会将写请求广播给 各个 Server，各个 Server 写成功之后会通知 Leader。
3. 当 Leader 收到此大多数 Server 写数据成功后，那么就认为数据写成功了。例如三个节点中有两个写成功了那么就认为写成功了。之后 Leader 会告诉 Server 写成功了。
4. Server 通知 Client 数据写成功了，写操作就完成了。