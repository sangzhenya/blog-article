## JPA EntityManager 简介

一个简单的使用 JPA 插入数据的过程如下所示：

```java
// 根据 persistence-unit 的信息创建 EntityManagerFactory
EntityManagerFactory factory = Persistence.createEntityManagerFactory("NewPersistenceUnit");
// 使用 EntityManagerFactory 创建 EntityManager
EntityManager manager = factory.createEntityManager();
// 使用 EntityTransaction 创建 EntityTransaction
EntityTransaction transaction = manager.getTransaction();

// 开启事务
transaction.begin();

// 准备要插入的数据
Book book = new Book();
book.setName("Test Book");
book.setPrice(12.5d);

// 使用 EntityManager 持久化对象
manager.persist(book);

// 提交事务
transaction.commit();

// 关闭 EntityManager 和 EntityManagerFactory
manager.close();
factory.close();
```

#### EntityManagerFactory

`EntityManagerFactory` 接口主要用来创建 `EntityManager` 实例，主要有以下几个接口：

1. `createEntityManager` 用于创建  `EntityManager` 实例。
2. `createEntityManager(Map map)` 同上，参数用于提供 `EntityManager` 属性。
3. `isOpen` 检查 `EntityManagerFactory` 是否处于打开状态，实体管理器工厂创建后一直处于打开状态，直到调用 `close` 方法将其关闭。
4. `close` 关闭 EntityManagerFactory。关闭后会释放所有资源，其他方法将不能调用，否则会抛出 `IllegalStateException` 异常。

#### EntityManager

在 JPA 规范中，EntityManager 是完成持久化操作的核心对象。实体对象只有在调用 EntityManager 将其持久化后才会变成持久化对象。EntityManager 对象在一组实体类与底层数据源之间进行 OR 映射管理，可以用来管理和更新 Entity Bean。实体对象主要有以下几种状态：

1. 新建状态：新创建的对象，尚未拥有持久性主键。
2. 持久化状态：已经拥有持久性主键并和持久化建立了上下文环境。
3. 游离状态：拥有持久化主键，但是没有与持久化建立上下文环境。
4. 删除状态：拥有持久化主键，已经和持久化建立上下文环境，但是已经从数据库删除。

EntityManager 主要有以下几个方法:

**find**

返回指定 ID 对应的实体类对象，如果对应的实体存在于当前的持久化环境，则返回一个被缓存的对象；否则会新建一个新的 Entity，并加载数据库中相关的信息；如果 ID 不存在于数据库中则返回一个 null。

**getReference**

与 find 方法类似，不同点是如果缓存中不存在则返回一个 Entity 的代理并不会立即加载数据，而是在第一次使用 Entity 的时候才会加载，所有如果 ID 不存在则直接回抛出 `EntityNotFoundException`。

**persist**

将新创建的 Entity 纳入到 EntityManager 的管理。在此方法执行之后，传入的 Entity 对象转换成持久化状态。如果参数已经是持久化对象则什么都不做，如果已经删除状态的 Entity 则会转换为持久化状态，如果是游离状态的实体，可能会抛出 `EntityExistsException` 异常。

**remove**

删除 Entity，如果 Entity 是被 EntityManager 管理的，则同时会删除关联的数据库记录。

**merge**

用于处理 Entity 的同步即数据库的插入和更新操作。具体流程如下：

![Merge 流程](http://img.sangzhenya.com/Snipaste_2019-12-21_15-28-47.png)

**flush**

同步持久化上下文环境，即将持久化环境所所有未保存的 Entity 的状态信息保存到数据库中。

**setFlushMode**

设置持久化上下文环境的 Flush 模式，有 AUTO 自动更新数据库实体和 COMMIT 为直到提交事务时才更新数据库记录两种模式选择。

**getFlushMode**

获取持久化上下文环境的 Flush 模式。

**refresh**

用数据库数据记录更新 Entity 对象状态属性值。

**clear**

清除持久化上下文环境，断开所有关联的 Entity，如果还有未提交的信息则会被撤销。

**contains**

判断一个 Entity 是否属于当前持久化上下文环境管理的 Entity。

**isOpen**

判断当前 EntityManager 实例是否打开状态。

**getTransaction**

获取资源层的事务管理对象，其可以用于开始和提交多个事务。

**close**

关闭 EntityManager，方法执行后若调用查询对象的方法会抛出 IllegalStateException 异常。如果当前 EntityManager 关联的事务处于活动状态，则持久化上下文会仍处于被管理状态直到事务完成。

**createQuery**

创建一个查询对象

**createNamedQuery**

创建查询对象，参数为命名的查询语句。

**createNativeQuery**

使用标准 SQL 语句创建查询对象。

**createNativeQuery(String sqlString, String resultSetMapping)**

使用标准 SQL 语句创建查询对象，并指定结果集 Map 的名称。



#### EntityTransaction

可以通过 EntityManager 的 getTransaction 方法获取其实例用于管理资源层实体管理器的事务操作。

**begin**

开启一个事务，此后多个数据库操作将作为一个整体被提交或者撤销。

**commit**

提交当前事务，即事务启动后所有数据库更新操作持久化到数据库中，

**rollback**

撤销当前事务，撤下事务启动后的所有数据库更新操作，从而不对数据产生影响。

**setRollbackOnly**

使当前事务职能被撤销

**getRollbackOnly**

查看当前事务是否设置了只能撤销的标志。

**isActive**

查看当前事务是否是活动的。