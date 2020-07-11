---
title: "MySQL 事务、视图、存储过程及 Join 查询"
tags: ["MySQL"]
categories: ["MySQL"]
date: "2019-05-19T09:00:00+08:00"
---

### 事务

对于 MySQL 中只有使用 Innodb 数据库搜索引擎的数据卷或表才支持事务，事务是用来维护数据库的完整性，保证成批的 SQL 语句要么全部执行，要么全部不执行。用来管理 insert, update, delete 语句，一般来说事务需要满足 4 个条件 ACID，即原子性 A，一致性 C，隔离性 I，持久性 D。

1. 原子性：一个事务的所有操作要么全部完成，要么全部不完成，不会结束在中间某个环境。事务在执行过程中发生错误，会被回归到事务开始前的状态，和事务从来没有执行过一样。
2. 一致性：在事务开始之前和事务结束之后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，着包含资料的精确度、串联性以及后续数据库可以自发的完成预定的工作。
3. 隔离性：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以放着多个事务并发的执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括：未提交、读提交、可重复读和串行化。
4. 持久性：事务结束之后，对数据的修改是永久有效的，即便是系统故障也不会丢失。

*需要注意的是 MySQL 命令行默认设置下，事务都是自动提交的，即执行 SQL语句之后就马上执行 Commit 操作。需要显示地开启一个事务需要使用 BEGIN 或者 START TRANSACTION，或者执行命令 SET AUTOCOMMIT=0 用来禁止使用当前会话的自动提交。*

对于嵌套事务的情况，可以使用 savepoint 实现，事务可以回滚到 savepoint 而不影响 savepoint 之前的变化不需要放弃整个事务。rollback 的时候可以指定 savepoint。使用 savepoint 的 删除 savepoint 可以使用如下指令：

```mysql
# 设置 savepoint 
savepoint savepoint_name;

# 回滚到执行的 savepoint
rollback to savepoint_name

# 删除 savepoint
release savepoint savepoint_name
```

隔离级别如下图所示：

![MySQL 隔离级别]( http://img.programya.com/Snipaste_2019-10-12_16-34-58.png )

### 视图

视图的本质就是一张虚拟表，一张逻辑表，其本身并不包含数据，是作为一个 select 语句保存在数据字典中，即本质是对一个查询的封装，虚拟的表，一旦封装的内容改变，视图的内容也会随之而改变。反之对视图的操作和对表的操作一样，可以对其进行查询，修改和删除。对于视图所看到的内容修改，相应的本表也会随之而修改，因为本质上是同一份数据。

视图有很多的优点。首先简单，使用视图的用户完全不用关心后面对应的表的结构，关联条件和筛选条件，对用户来说是过滤好的复合条件的合集。其次安全，使用视图的用户只能访问被允许查询的结果集，对表的权限管理并不能限制到具体的行列，通过视图则可以简单的实现。最后是数据独立，一旦视图的结构确定了，可以屏蔽表结构的变化对数据的应许，源表增加列队视图没有影响。源表修改列名可以通过修改视图来解决，不会造成对访问者的影响。

创建视图可以使用以下的 SQL 语句

```mysql
CREATE [OR REPLACE] [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    VIEW view_name [(column_list)]
    AS select_statement
   [WITH [CASCADED | LOCAL] CHECK OPTION]
# create  or repleace 创建或者替换
# algorithm 设置视图选择的算法,默认是 undefined 表示自动选择，merge 表示合并，template 表示临时表
# with 后设置视图在更新的时候保证在视图的权限范围之内，例如 cascade 是默认值，表示更新视图的时候要满足
# 视图和表的相关条件，而 local 表示更新视图的时候满足视图定义的条件即可。

# 例如
create view v_user_view (id, name, gender, age)
as select user.id, user.name, user_detail.gender, user_detail.age from user, user_detail
where user.id = user_detail.user_id where age > 18
with check option;

# 查看视图信息
show create view v_user_view;

# 查看视图内容
select * from v_user_view;

# 修改视图
create or replace view V_user_view as [select sql]

ALTER
    [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
    [DEFINER = { user | CURRENT_USER }]
    [SQL SECURITY { DEFINER | INVOKER }]
VIEW view_name [(column_list)]
AS select_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION]

# 删除视图
DROP VIEW [IF EXISTS]  view_name [, view_name]
```



### 存储过程

存储过程类似于函数，即是把一段代码封装起来，当要执行着一段代码的时候，可以通过相应的存储过程实现，在封装的语句体里面可以用分支和循环结构。存储过程仅在创建的时候进行编译，以后每次执行的时候不会再次编译，可以提高数据库执行速度。当对数据库进行复杂的操作的时候，可以将复杂操作用存储过程封装与数据提供的事务处理结合一起使用。存储过程可以重复使用进而减少数据库开发人员的工作量。安全性高可以设定只有某些用户才具有对指定存储过程的使用权。使用简单只需要指定存储过程的名字和参数就可以使用。

```mysql
# 创建存储过程
delimiter $$
create procedure procedure_name([arg_name arg_type, ])
begin
#...
end $$
delimiter ; 

# 查看现有的存储过程
show procedure status;

# 删除存储过程
drop procedure procedure_name;

# 调用存储过程
call procedure_name([args])
```



### Join 查询

![七种 Join](http://img.programya.com/WcPXlpgk4fIHT2w.png)

1. 左连接 left join
2. 右连接 right join
3. 内连接 inner join
4. 左独连接 left join excluding inner join
5. 右独连接 right join excluding inner join
6. 全连接 full join
7. 全连接去交集 full join excluding inner join

