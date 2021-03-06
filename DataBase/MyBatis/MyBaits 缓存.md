---
title: "MyBatis 缓存"
tags: ["MyBaits"]
categories: ["MyBaits"]
date: "2019-05-07T09:00:00+08:00"
---

MyBatis 具有非常强大的缓存特性，可以方便的配置和定制。其默认定义了两级缓存：分布是一集缓存和二级缓存。

![缓存示意图](http://img.programya.com/20191228112730.png)

一级缓存：又称本地缓存。作用域为 sqlSession, 当 Session flush 或 close 之后，该 Session 中的所有 Cache 将被清空。其默认情况一级缓存开启，而且不能关闭，不过可以通过调用 `clearCache` 来清空本地缓存可以通过设置 `localCacheScope` 来修改缓存的作用域，有 SESSION | STATEMENT 两种。默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。同一次 Session 期间只要查询过的数据都会被保存在当前 Sql Session 的一个 Map 中，key 是 hashCode + sqlId + 查询 sql 的语句+ 参数。其会再以下四种情况下会失效：

1. 不同 Sql Session 之间的查询。
2. 同一个 Sql Session 查询条件不同时。
3. 同一个 Sql Session 但是在两个查询之间执行了增删改操作。
4. 同一个 Sql Session 两次查询之间手动清空了缓存。

二级缓存是基于 namespace 级别的缓存，不同的 namespace 对应不同的缓存 map。二级缓存需要手动配置开启，其还要求 POJO 实现 `Serializable` 接口。二级缓存在 Sql Session 提交或者关闭之后才会生效。开启需要以下两个步骤：

1. 在配置中设置 `cacheEnabled` 为 `true`。
2. 在许 cache 缓存的地方加上 `<cache />` 配置。

配置如下：

```xml
<mapper namespace="com.xinyue.imybatis.mapper.UserMapper">
    <cache eviction="LRU" flushInterval="360000" />
    <!-- eviction 缓存策略，有 LRU 最少使用；FIFO 先进先出；SOFT 软引用；WEAK 弱引用四种方式，默认是 LRU
 				flushInterval 刷新间隔配置的是毫秒数；size 缓存中存放多少元素；type 指定自定义缓存的全类名，需要实现 Cache 接口；
				readOnly 是否只读，true 表示从缓存中获取的操作只是读的操作不会修改数据，会直接将引用交给用户。false 表示可能会修改，则会使用序列化和反序列化的方式克隆一份新的数据
 				-->
</mapper>
```

与缓存相关的配置还有以下几点：

`select` 标签上的 `useCache` 属性，配置当前查询是否使用二级缓存。

`sql` 标签上的 `flush` 属性，增删改默认为 `true`，查询默认为 `false`。sql 执行完之后会同时清空一二级缓存。

当然也可以使用第三方的 Cache ，例如 EhCache 首先需要在 pom 文件中加出来相关的依赖：

```xml
<!-- https://mvnrepository.com/artifact/org.ehcache/ehcache -->
<dependency>
  <groupId>org.ehcache</groupId>
  <artifactId>ehcache</artifactId>
  <version>3.8.1</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.mybatis.caches/mybatis-ehcache -->
<dependency>
  <groupId>org.mybatis.caches</groupId>
  <artifactId>mybatis-ehcache</artifactId>
  <version>1.1.0</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-simple -->
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-simple</artifactId>
  <version>1.7.30</version>
  <scope>test</scope>
</dependency>
```

之后指定 `cache` 的 `type` 属性为 `org.mybatis.caches.ehcache.EhcacheCache` 。

最后编写 `ehcache` 的配置文件：

```xml
<ehcache>
 <!-- cache 保存路径 -->
 <diskStore path="~/temp/ehcache" />
 <defaultCache 
   maxElementsInMemory="10000" 
   maxElementsOnDisk="10000000"
   eternal="false" 
   overflowToDisk="true" 
   timeToIdleSeconds="120"
   timeToLiveSeconds="120" 
   diskExpiryThreadIntervalSeconds="120"
   memoryStoreEvictionPolicy="LRU">
 </defaultCache>
</ehcache>
```

其中 `defaultCache` 是配置 `ehcache` 的管理策略：

1. `maxElementsInMemory` 内存缓存中元素的最大数目，必须
2. `maxElementsOnDisk`  磁盘缓存中元素的周达数目，0 表示无限制，必须
3. `eternal` 设置元素是否是永不过期。如果为 `true`，则缓存的数据始终有效；如果为 `false` 那么还要根据 `timeToIdleSeconds`，`timeToLiveSeconds` 判断，必须
4. `overflowToDisk` 设置当前内存缓存溢出的时候是否将过期的元素缓存到磁盘上，必须
5. `timeToIdleSeconds` 当缓存在 `EhCache` 中数据前后两次访问时间超过 `timeToIdleSeconds` 的属性值时，缓存数据会被删除；默认为 0 即无限
6. `timeToLiveSeconds` 缓存元素的有效生命周期，默认为 0 即无限
7. `diskSpoolBufferSizeMB` 设置磁盘缓存的缓存区大小，默认是 30M，每个 Cache 都应该有自己的缓冲区
8. `diskPersistent` 在 VM 重启的时候是否启用磁盘保存 `EhCache` 中的数据，默认是 `false`
9. `diskExpiryThreadIntervalSeconds` 磁盘缓存的清理线程运行间隔，默认是 120s，相应的线程进行一次 EhCache 中数据的清理
10. `memoryStoreEvictionPolicy` 当缓存达到最大，有新元素加入的时候，移除缓存侧策略。默认为 LRU，也可以选择 LFU 和 FIFO 两种



