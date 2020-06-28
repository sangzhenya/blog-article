## 缓存

[toc]

### JSR 107

首先 JSR 107 即 JCache API的首个早期草案，在 Java JCache 中定义了 5 个核心内容，即 CachingProvider，CacheMaager，Cache，Entry 和 Expiry。

1. CachingProvider 中定义了创建、配置、获取、管理和控制多个 CacheManager, 一个应用可以在运行期间访问多个 CahingProvider。
2. CacheManger 定义了创建、配置、获取和控制多个唯一命名的 Cache，这些 Cache 存在于 CacheManger 的上下文中。一个 CacheManger 仅被一个 CacheProvider 所拥有。
3. Cache 是一个类似 Map 的数据结构并临时存储以 key 为索引的值，一个 Cache 仅被一个 CacheManger 所拥有。
4. Entry 是一个存在于 Cache 中 key - value 对
5. Expiry 每个存储在 Cache 中的条目有一个定义的有效期，一旦超过这个事件，条目为过期状态。一旦过期，条目将不可访问，更新或者删除，缓存的有效期有 ExpiryPolicy 设置。

结构如下图所示：

![结构图](https://i.loli.net/2020/06/24/1LHzeNhpbtrQoxu.png)

### Spring Cache

首先通过 `@EnableCaching` 开启基于注解的缓存，Spring 主要提供了两个核心抽象：

1. `Cache` 缓存接口，定义缓存操作常见的实现有： `RedisCache`, `EhCacheCache`, `ConcurrentMapCache` 等等。
2. `CacheManager` 缓存管理器，管理各种缓存组件。

主要结构图如下：

![结构图](https://i.loli.net/2020/06/28/OFqY8Mcampnv9DX.png)

主要的缓存注解有以下几个：

1. `@Cacheable` 针对于方法的设置，能够根据方法的请求参数对其结果进行缓存
   1. `cacheNames/value` 设置放到 cache 的名称，可以设置多个 `{"books", "isbns"}`
   2. `key` 缓存的 key，如果不指定则默认使用方法的值，如果只有一个参数则直接使用该参数，否则会将所有参数封装成 `SimpleKey` 作为 key。
   3. `keyGenerator` 设置 key 生成的规则
   4. `cacheManager` 多个 CacheManager 的指定使用哪个 CacheManager
   5. `cacheResolver` 设置缓存解析器，和 CacheManager 的作用类似，可以动态的决定放到哪个 cache 中，
   6. `condition` 在特定情况下缓存
   7. `unless` 否定缓存，指定的条件不缓存
   8. `sync` 是否采用异步的方式 
2. `@CacheEvict` 清空缓存
3. `@CachePut` 保证方法被调用，又希望结果被缓存
4. `@CacheConfig` 类级别的 common 的注解配置

### 缓存的原理

#### 自动配置的原理

首先通过 `CacheAutoConfiguration` 缓存配置类导入 `CacheConfigurationImportSelector` 和 `CacheManagerEntityManagerFactoryDependsOnPostProcessor` 会为容器导入 `GenericCacheConfiguration`， `RedisCacheConfiguration`，`SimpleCacheConfiguration` 等配置类。默认是 `SimpleCacheConfiguration`，会为容器中注入一个 `ConcurrentMapCacheManager` 即一个 CacheManager, 其 getCache 方法即通过一个名字获取一个缓存组件。如果存在则直接获取，否则则会去创建一个 cache。其将所有数据存储到一个 `ConcurrentMap` 中（`ConcurrentMapCache`）。

```java
@Override
public Cache getCache(String name) {
		Cache cache = this.cacheMap.get(name);
		if (cache == null && this.dynamic) {
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
					cache = createConcurrentMapCache(name);
					this.cacheMap.put(name, cache);
				}
			}
		}
		return cache;
	}
```

```java
protected Cache createConcurrentMapCache(String name) {
		SerializationDelegate actualSerialization = (isStoreByValue() ? this.serialization : null);
		return new ConcurrentMapCache(name, new ConcurrentHashMap<>(256),
				isAllowNullValues(), actualSerialization);

	}
```

#### 运行的步骤

