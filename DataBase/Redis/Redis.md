---
title: "Redis"
tags: ["Redis"]
categories: ["Redis"]
date: "2019-05-25T09:00:00+08:00"
---

### NoSQL

NoSQL 即 Not Only Sql, 不仅仅是 SQL。泛指非关系型数据库。

1. 易扩展

   NoSQL 数据库种类繁多，共同点都是去掉了关系数据库的关系型特性。数据之间无关系，这样就非常容易扩展。

2. 大数据量高性能

   NoSQL 数据库都具有非常高的读写性能，主要得益于其无关性，数据库结构简单。

3. 多样灵活的数据模型

   NoSQL 无需事先为要存储的数据建立字段，随时可以存储自定义的数据格式。而这个是在关系数据库中是非常麻烦的。


NoSQLPL 主要有以下四种分类：

1. KV 键值：Redis
2. 文档型数据库：MongoDB
3. 列存储数据库：HBase
4. 图关系数据库：Neo4J

### Redis

Redis 即 Remote Dictionary Server  是使用 C 预言开发，完全开源免费的一个高性能的 key/value 分布式数据库，基于内存运行，且支持持久化的 NoSQL 数据库。Redis 支持数据的持久化可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用；支持多种数据结构；支持数据备份，主从复制。

Redis 是单进程的，通过对 epoll 函数的包装来做到对读写事件的响应。其默认有 16 个数据库，初始默认使用 0 号库。常用的几个命令如下：

```shell
# 切换数据库
select 0
# 查看当前数据库的 key 的数量
dbsize
# 清空当前库
flushdb
# 清空全部库
flushall
```



### Redis 数据类型

总共有 5 中数据类型：String, Hash, List, Set, ZSet

#### String

是最基本的数据类型，和 Memcached 一样，一个 key 对应一个 value。String 是二进制安全的，即可以保存任何数据（例如 图片或者序列化的对象）。String 的 value 最大是 512M。

#### Hash

类似于 Java 中的 Map<String, Object>，是一个 String 类型的 field 和 value  的映射表。

#### List

是简单的字符串列表，按照插入顺序排序。可以添加一个元素到列表的头部或者尾部，本质是一个链表。

#### Set

是字符串的无序不重复集合，通过 HashTable 实现的。

#### Zset

和 Set 一样是一个字符串不重复集合，但允许每个元素关联一个 double 类型的分数， redis 会通过分为集合中的成员进行从小到达排序。



常用的相关命令如下：

**key 相关**

```shell

# 查看所有的 Key
key *
# 判断某个 key 是否存在
exists key
# 将 key 移动到某个 Db
move key db
# 为 key 设置过期时间
expire key seconds
# 查看 key 还有多少秒过期，-1 表示永不过期， -2 表示已经过期
ttl key
# 查看 key 的类型
type key
```

**String 相关**

```shell
# 设置/获取/删除/增加/获取长度
setget/del/append/strlen
# +1/-1/增加 n/减少 n （只能数字类型）
incr/decr/incrby/decrby
# 获取字符串子串值 -1 表示全部
getrange
# 修改字符串子串
setrange
# 设置一个 seconds 过期的值
setex key seconds value
# 不存在则设置
setnx key value
# 先get 后 set
getset key value
```

**Hash 相关**

```shell
# 设置/获取/获取某个/获取all/删除某个
hset/hget/hmget/hgetall/hdel
# 获取 size
hlen
# 是否存在
hexists key
# keys/values
hkeys/hvals
# 增加
hincrby/hincrbyfloat
# 不存在则设置
hsetnx
```

**List 相关**

```shell
# 从左添加/从右添加/获取指定范围
lpush/rpush/lrange
# 左弹出/右弹出
lpop/rpop
# 通过 index 获取下标
lindex key index
# 获取长度
llen key
# 删除 N 个 value
lrem key N value 
# 截取 list
ltrim key 
# list 直接的数据迁移
rpoplpush 
# 设置指定 index 的 value
lset key index value
# 在某个value之前或者之后添加值
linsert key before/after value
```

**Set相关**

```shell
# 设置/获取/判断
sadd/smembers/sismeber
# 获取元素个数
scard
# 删除集合中的元素
srem key value
# 随机出几个数
srandmember key n
# 随机出栈
spop key
# 数据迁移
smove key1 key2
# 差集/并集/交集
sdiff/sunion/sinter
```

**Zset 相关**

```shell
# 设置和获取
zset/zrange
# 获取指点范围的 key
zrangebyscore key
# 删除某个 score 的下的元素
zrem key
# 数量/一定范围的数量/指定 key 的分数
zcard/zcount key/zrank key 
# 反序排列
zrevrange
# 一定范围的反序排列
zreangebyscore key
```



### Redis 配置

`redis.conf` 中的常用配置：

```properties
# 单位如下
# 1k => 1000 bytes
# 1kb => 1024 bytes
# 1m => 1000000 bytes
# 1mb => 1024*1024 bytes
# 1g => 1000000000 bytes
# 1gb => 1024*1024*1024 bytes
#
# units are case insensitive so 1GB 1Gb 1gB are all the same.

# 引入其他 conf
# include /path/to/local.conf

# 守护进程模式
daemonize yes
# 守护进程模式下的 pid
pidfile /var/run/redis_6379.pid
# 端口
port 6379
# 设置 tcp 的 backlog，其backlog 是一个连接队列。
# backlog 队列总和 = 未完成三次握手队列 + 已经完成三次握手队列
# 在高并发环境下需要一个高的 backlog 值去避免客户端连接的问题。
# Linux 内核会将这个值减小到 /proc/sys/net/core/somaxconn 的值。
# 所以需要确认增大 somaxconn 和 tcp_max_syn_backlog 两个值进而达到想要的效果
top-backlog 1511
# 绑定地址
# bind 127.0.0.1
# 超时时间，空闲 n 秒之后关闭连接， 0 表示禁用
timeout 0
# keepalive 检测时间。0 表示禁用
tcp-keepalive 0
# 日志级别 (debug, verbose, notice, warning)
loglevel notice
# 日志文件
logfile ""
# 系统日志
#syslog-enabled no
# 系统日志标志
#syslog-ident redis
# syslog 设备
#syslog-facility local0
# 数据库数量
databases 16
# 设置密码：每次命令前都需要密码验证
# requirepass foobared
# 最大客户端数量
# maxclients 10000
# 最大内存
# maxmemory <bytes>
# 最大内存策略
# maxmemory-policy  noeviction 
# 设置样本数量， LRU 算法和最小 TTL 算法都并非是精确算法，而是估算值。
# maxmomory-samples 5
```

缓存过期策略：

1. volatile-lru: 对设置了过期时间的 key 使用 LRU 算法移除。
2. allkey-lru: 使用 LRU 算法移除 key
3. volatile-random:  在过期的集合中随机删除 key。
4. allkeys-random: 随机删除 key
5. volatile-ttl: 移除 ttl 值最小的 key
6. noevictioin: 不进行移除操作。对写操作返回错误信息。

