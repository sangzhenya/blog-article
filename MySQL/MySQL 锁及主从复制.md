### MySQL 锁及主从复制

### 锁

从数据操作的类型可以分为读锁和写锁，从对数据操作的粒度分可以分为表锁和行锁。

#### 读写锁

读锁：共享锁针对同一份数据，多个读操作可以同时进行而不会互相影响。

写锁：排它锁，当前写操作没有完成前会阻断其他写锁和读锁。

#### 行表锁

表锁: 偏读，偏向于 MyISAM 存储引擎，开销小，加锁快；锁粒度大，发生锁冲突的概率高，并发读低。

通过命令 `show open tables` 可以查看哪些表被加锁，通过 `show status like 'table%';` 命令查看表锁定概况。其中 `Table_locks_immediate` 产生表级锁定的次数，可以立即获取锁的查询次数，每次立即获取锁值加 1， `Table_locks_waited` 出现表级锁定争用而发生等待的次数，每等待一次锁值加1。

![表锁定概况](http://img.sangzhenya.com/Snipaste_2019-10-20_10-52-59.png)

通过 `lock table tableA read;` 为表 tableA 加上 读锁。当前 session 在加锁之后，解锁之前，不能操作执行插入或更新操作，同时也不能操作未被当前锁锁的表，例如操作 tableB。其他session 可以读tableA，但是如果尝试更新 tableA 则会进入到等待状态，直到当前 session 使用 `unlock tables` 解除当前表的锁。

通过 `local table tableA write;` 为表 tableB 加上写锁。 当前 session 在加锁之后，解锁之前不能操作未被当前锁锁的其他表。其他 session 尝试查询，插入或者更新 tableA 的时候回进入等待状态，直到当前 session 使用 `unlock tables` 解除当前表的锁。



行锁: 偏写，偏向于 InnoDB 存储引擎，开销大，加锁慢；会出现死锁，锁粒度小，发生锁冲突概率低，并发度高。

通过命令 `show status like 'innodb_row_lock%';` 查看行锁使用情况。其中 `Innodb_row_lock_current_waits` 当前正等待锁定的数量；`Innodb_row_lock_time` 从系统启动当现在锁定总时间长度；`Innodb_row_lock_time_avg` 每次等待所花的平均时间；`Innodb_row_lock_time_max` 从系统启动到现在等待最长的一次所花的时间；`Innodb_row_lock_waits` 系统启动到现在总共等待的次数。

![行锁使用情况](http://img.sangzhenya.com/Snipaste_2019-10-20_11-22-50.png)

在默认的隔离级别  REPEATABLE-READ 的情况下（可以通过 `show variables like 'tx_isolation';` 查看隔离级别），在更新 SQL 执行之后，commit 之前对当前行的数据都持有行锁。此时其他 session 操作同一行的数据会进入等待状态，直到超时或者当前 session 执行 commit 操作。此时也只有当前 session 可以读取到更新的数据，其他 session 只能读取当旧的数据。

此外索引使用不当的情况下可能会导致行锁升级为表锁，例如 where 之后索引条件值未加单引号导致索引失效进而导致行锁升级为表锁。

间隙锁，对于使用范围条件而不是相等条件检索数据并请求共享或者排他锁时， InnoDB 会给符合条件的已有数据记录的索引项加上锁，对于键值在条件范围但并不存在的记录叫做间隙，InnoDB 也会对这个间隙加锁，这种加锁机制就是所谓的间隙锁（next-key 锁）。

可以使用 `select  * from tableA where col1='xxx' for update;` 对某一行进行加锁，直到执行 commit 操作。

尽量让索引数据检索都通过索引完成，避免无索引行锁升级为表锁。合理设计索引，尽量缩小锁的范围，使用较少的检索条件，避免间隙锁。尽量控制事务大小，减少锁定资源和时间长度。尽可能降低事务隔离级别。

### 主从复制