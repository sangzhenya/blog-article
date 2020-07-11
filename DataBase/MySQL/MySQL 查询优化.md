---
title: "MySQL 查询优化"
tags: ["MySQL"]
categories: ["MySQL"]
date: "2019-05-17T09:00:00+08:00"
---

### 架构

MySQL 架构图如下：

![MySQL 架构图](http://img.programya.com/Snipaste_2019-10-19_21-50-19.png)

#### 连接层

最上层是一些客户端和连接服务，包含本地 Socket 通信和大多数基于客户端/服务端工具实现的类似 TCP/IP 的通讯。主要完成一些类似于连接处理、授权认证及相关安全的方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层是基于 SSL 的安全连接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

#### 服务层

第二层架构完成了大多数的核心服务功能，例如 SQL结构，缓存查询，SQL 的分析及优化及部分内置函数的执行。所有跨存储引擎的功能也在这一层实现，例如存储过程和函数等，在该层，服务器会解析查询并创建相应的内部解析树，并对其完成相应的优化例如确定查询表的顺序，是否利用索引等，最后生成的相应的执行操作。如果是 SELECT 语句，那么服务器还会查询内部缓存，如果缓存空间足够大可以显著提高大量读操作的系统性能。

#### 引擎层

存储引擎是真正的负责了 MySQL 的的数据存储和读取，服务器通过 API 与存储引擎进行通信。不同的存储引擎具有的功能不同，可以根据自己实际的需求更改，最常用的是 MyISAM 和 InnoDB。

#### 存储层

数据存储层主要是将数据存储在运行于裸设备文件系统之上，并完成与存储引擎的交互。

### 存储引擎

在 MySQL 中可以查看具体使用的存储引擎。

```mysql
-- 查询索引支持存储引擎
show engines;
-- 查询使用的存储引擎
show variables like '%storage_engine%';
```

![查询使用的存储引擎](http://img.programya.com/Snipaste_2019-10-19_22-12-47.png)

MyISAM 和 InnoDB 的对比：

| 对比   | MyISAM     | InnoDB                                                       |
| ------ | ---------- | ------------------------------------------------------------ |
| 主外键 | 不支持     | 支持                                                         |
| 事务   | 不支持     | 支持                                                         |
| 行表锁 | 支持到表锁 | 支持到行锁，适合高并发操作                                   |
| 缓存   | 只缓存索引 | 缓存索引还有缓存真实数据，对内存要求较高，而且内存大小对性能有决定性的影响 |
| 表空间 | 小         | 大                                                           |
| 关注点 | 性能       | 事务                                                         |

### 配置及日志

Windows 环境下默认配置文件是在 MySQL 安装目录下的 `my.ini` 文件。

数据库编码设置：可以通过 `show variables like '%char%';` 命令查看当前数据的字符集编码方式设置情况。

![数据库字符集](http://img.programya.com/Snipaste_2019-10-20_08-47-30.png)

可以通过配置以上的参数设置数据库的默认编码：

````properties
[client]
default-character-set=utf8

[mysqld]
character_set_server=utf8
character_set_client=utf8
collationi-server=utf8_general_ci

[mysql]
default-character-set=utf8
````

二进制日志：log-bin 配置主要用于 主从复制，例如 `log-bin=D:/data/mysqlbin`

错误日志：log error 记录错误日志，记录严重的警告和错误信息，每次启动和关闭的详细信息等，例如 `log-err=D:/data/mysqlerr`

查询日志：默认是关闭的，记录查询的 SQL 语句，开启会降低 MySQL 的整体性能。例如 `general_log=1`, `general_log_file=D:/logs/xinyue-general-log`, `log_output=FILE`。同样也可以直接通过命令 `set global general_log=1;`， `set global log_output='TABLE';` 命令行的方式开启，此种方式可以通过 `select * from mysql.general_log;` 查询 SQL 记录。

数据文件：默认在 data 下面会有每个数据库对应的一个文件夹。其中 frm 文件存放表结构；myd 文件存放表数据；myi 文件存放索引。

### 查询优化

1. 小表驱动大表： 子查询中中标尽量是小表，如果小表驱动大表则使用 in 优于使用 exists，反之则 exist 优于 in。exist 可以理解为将主查询数据放到子查询做条件验证，根据验证结果来决定主查询的数据结构是否得以保留。
2. Order By 子句尽量使用 Index 方式排序，[结合where字句]遵循索引建的最佳左前缀，避免使用 filesort 方法排序。对于 filesort 有双路排序和单路排序两种，其中 Query 字段大小总和小于 `max_length_for_sort_data`且排序字段不是 text，blob 类型时会使用单路排序算法，否则还是使用老的双路排序算法。另外提高 `sort_buffer_size` 可以提高效率，但要根据实际需要设置。
3. Group By 本质上是先排序后分组，同样是尽量使用索引列，且遵循索引建的最佳左前缀，且优先使用 where，能使用 where 后的尽量不要使用 having 限制。当无法使用索引列时，可以适当增大 `max_length_for_sort_data` 和 `sort_buffer_size` 参数设置。

### 慢日志查询

MySQL 的慢日志是 MySQL 提供的一种日志记录，它用量记录在 MySQL 中响应时间超过阈值的语句，即运行时间超过 `long_query_time` 值的 SQL，该值默认是 10，即运行在 10s 以上的语句。默认情况是不开启的，需要手工设置。通过 `show variables like '%slow_query_log%';` 查看慢日志状态，命令行中通过 `set gloable slow_query_log = 1;` 开启慢日志，仅针对当前数据的本次运行开启。可以在 `my.ini` 中配置永久开启。通过 `show global status like '%slow_queries%';` 可以查询 slow sql 的数量。此外 MySQL 提供了 mysqldumpslow 工具用于更好代分析慢日志文件。

![慢日志状态](http://img.programya.com/Snipaste_2019-10-20_09-44-15.png)

```properties
[mysqld]
# 开启慢日志
show_query_log=1
# 设置慢日志存放位置
show_query_log=D:\logs\xinyue-slow.log
#设置超时时间 > 
long_query_time=5
log_output=FILE
```



### show profile 分析

show profile 是 MySQL 提供的可以用来分析当前会话中语句执行的资源消耗情况，用于 SQL 调优的测量。默认是关闭的，开启后默认保存最近 15 的结果。可以通过 `show variables like 'profiling';` 命令查看状态，通过 `set profiling=on;` 开启，同样可以在 配置文件中设置永久开启。通过命令 `show profiles;` 命令查看运行的命令。

![show profile 命令](http://img.programya.com/Snipaste_2019-10-20_10-08-27.png)

通过 `show profile cpu, block io for query [queryId];` 具体分析 SQL 执行情况。	其中主要有以下参数：

1.  ALL 所有开销信息
2. Block IO 块 IO 相关开销
3. Context Switches 上下文切换相关开销
4. CPU  CPU相关开销
5. IPC 发送和接收相关开销信息。
6. Memory 内存相关开销信息
7. Page Faults 页面错误相关开销信息
8. Source  和 source_function, source_file, source_line 相关的开销信息
9. Swaps 交换次数相关开销