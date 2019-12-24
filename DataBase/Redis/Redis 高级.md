## Redis 高级

### Redis 持久化

#### RDB

Redis DataBase，在指定的时间间隔内将内存中的数据集快照写入磁盘，恢复的时候是将快照文件直接读到内存中。Redis 会单独的创建一个子进程来进行持久化，会先将数据写入到一个临时传文件中，等待持久化过程结束时在用这个临时文件替换上次持久化号的文件。整个过程中主进程是不进行任何 IO 操作的，这就确保了极高的性能。

如果需要进行大规模数据的恢复，而且对于数据恢复的完整性不是非常敏感，那 RDB 方式要比 AOF 方式更加的高效。 RBD 缺点是最好一次持久化的数据可能丢失。

复制一个与当前线程一样的进程，新进程所有的数据（变量，环境变量，程序计数器等）数据值和原进程一致，但是是一个全新的进程并作为原进程的子进程。

RBD 保存的是 dump.db 文件，在 `redis.conf` 中的

```properties
save 900 1
save 300 10
save 60 10000
```

几个关于持久化的配置

```properties
# 在后台出错的时候停止写操作，默认是 yes
stop-writes-on-bgsave-error yes
# 压缩备份文件
rdbcompression yes
# 启用数据校验
rdbchecksum yes
# 备份文件名称
dbfilename dump.rdb
# 目录
dir
```

手动触发持久化

`save/bgsave/flushall` 都会触发持久化操作。

`redis-cli config set save ""` 动态关闭 RDB 保存规则

#### AOF

Append Only File，以日志的形式来记录每个写操作，将 Redis 执行过的所以写指令记录下来，只允许追加文件但是不可以改写文件，redis 启动之初会读取改文件重新构建数据，即 redis 启动的话就根据日志的内容将写执行从执行一次以完成数据的回复工作。

```properties
# 默认是关闭状态
appendonly no
# 默认备份文件名称
appendfilename "appendonly.aof"
```

`aof `  文件出错的时候会导致 redis 启动失败，可以使用 `redis-check-aof --fix appendonly.aof` 修复受损的 `aof` 文件。

几个常用的配置：

```properties
# Always 同步持久化，每次变更数据会立即记录到磁盘，性能差但完整性好。
# Everysec 默认设置，异步操作，每秒记录，可能有数据丢失。
# No
appendfsync everysec
# 重启时是否运用 appendfsync 默认 no
no-appendfsync-on-rewrite
# 设置重写的基准值, 最小是64M，每次增加 1倍
auto-aof-rewrite-min-size 64mb
auto-aof-rewrite-percentage 100
```

AOF 采用文件追加方式，文件会越来越大为了避免出现此种情况，增加了重写机制，当 AOF 文件的大小超过所设定的阈值的时候，Redis 就会启动 AOF 文件的内容压缩，只保留可以恢复数据的最小指令集，可以使用 `bgrewriteaof ` 指令触发。具体就是 AOF 文件持续增长而过大的时候，会 fork 出一个新的线程来将文件重写（先写后改名），遍历新进程的内存中数据，每条记录都有一个 set 语句，重写 aof 文件的操作，不是读取旧的 aof 文件，而是将整个内存中的数据库内容使用命令的方式重写了一个新的 aof 文件。触发条件是上次重写 AOF 的大小，默认是当 AOF 文件大小是上次的 rewrite 后大小的一倍且文件超过 64M 时触发。

#### AOF 和 RDB 选择

RDB 持久化的方式能够在指定的时间间隔对你的数据进行快照存储。AOF 持久化则是记录每次对服务器的写操作，当服务器重启的时候会重新执行这些命令来回复原始数据，AOF 命令以 redis 协议追加保存每次写操作到文件末尾。 Redis 还能对 AOF 文件进行重写进而避免 AOF 文件的体积过大。如果只是做缓存服务器可以不使用任何持久化方式。在两种都开启的时候 redis 启动的时候会优先加载 AOF 文件来回复原始数据，通常情况写 AOF 文件保存的数据要比 RDB 文件保存的数据要完整。

RDB 备份可以做后备用途，建议在 Slave 上持久化 RDB 文件，而且保留 `save 900` 一条即可，如果是 启用了 AOF，好处是尽最大可能的避免了数据的丢失，坏处一是持续的 IO，二是 AOF rewrite 过程中产生的数据写到新文件会造成阻塞，在硬盘许可的情况下尽量避免 AOF rewrite 的频率，AOF 重写的默认值最好适当增大一些。如果不适用 AOF，可以依靠主从复制实现高可用，可以节省 IO 和 rewrite 带来系统开销。

### Redis 事务

Redis 事务可以一次性执行多个命令，即一组指令。一个事务中的所有命令都会序列化，按照顺序的串行化执行而不被其他命令插入。

```shell
# 取消事务
discard
# 执行事务
exec
# 标记事务开始
multi
# 取消 watch
unwatch
# 为 一个或多个 key 添加 watch，如果这些 key 被其他命令改动则事务被打断。
watch key
```

三个特性：

1. 单独的隔离操作：事务中的所有的命令都会序列化、按顺序的执行。事务执行的过程中，不会被其他客户端发送的命令请求所打断。
2. 没有隔离级别：队列中的命令没有被提交之前不会有任何实际执行，不存在可见性的问题。
3. 不保证原子性：Redis 一个事务中如果有一条命令执行是的，其他命令可能仍然被执行，没有回滚，例如 `incr` 指令失败。

### Redis 发布订阅

进程之间的一种消息通讯模式：发送者发送消息，订阅者接收消息，订阅发布模式。

```shell
# 订阅一个或多个符合格式的频道
PSUBSCRIBE pattern [pattern ...]
# 查看订阅与发布系统状态
PUBSUB subcommand [argument [argument ...]]
# 将信息发送到指定的频道
PUBLISH channel message
# 退订所有给定模式的频道
PUNSUBSCRIBE [pattern [pattern ...]]
# 订阅给定的一个或多个频道的信息
SUBSCRIBE channel [channel ...]
# 指退订给定的频道
UNSUBSCRIBE [channel [channel ...]]
```

### Redis 主从复制

主从复制 master/slaver 机制，master 以写为主，slave 以读为主。主要为了读写分离和容灾备份。

主要配置从机 `slaveof 主IP 主端口` ，通过 `info replication` 命令查看状态。在一主多仆的情况下，默认主写，从读。如果是通过命令设置主从关系，主 down 之后其余角色不变，主重启之后关系恢复正常。从机 down 之后重启默认为 master 需要重新和主机连接。也可以通过 `slave no one` 命令停止与其他数据的同步转为主数据库。

复制的具体原理是：Slave 启动成功连接 master 之后会发送 sync 命令，Master 接到命令后启动后台进程，同时收集所有接收到修改数据集的命令，在后台执行完毕后，master 将传送整个数据文件的 slave 完成一次完全的同步。Slave 服务在接收到数据文件后将其持久化并加载到内存中。Master 继续将新的所有收集到的修改命令依次传给 slave 完成同步，只要重新连接 master 都会进行一次完全的同步。

#### 哨兵模式

能够自动的监控主机否是故障，如果故障则根据投票数自动将从库转变为主库。增加 `sentinel.conf` 文件，在文件中添加 `sentinel monitor 主库名称  主IP  主端口 票数阈值`。然后通过 `redis-sentinel sentinel.conf` 启动哨兵。