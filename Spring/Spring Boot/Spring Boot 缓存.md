---
title: "Spring Boot 缓存"
tags: ["Spring", "Spring Boot"]
categories: ["Spring"]
date: "2019-02-21T09:00:00+08:00"
---

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
2. `@CacheEvict` 清空缓存, 其既可以从某一个或多个 cache 删除指定的 key，也可以通过 `allEntries` 指定删除 cache 中所有的条目。此外默认清除缓存是在方法之后执行的，不过其还可以通过 `beforeInvocation` 属性指定是否在方法执行之前执行删除操作。
3. `@CachePut` 保证方法被调用，又希望结果被缓存
4. `@CacheConfig` 类级别的 common 的注解配置

如果使用了 Spring Boot 则直接在 pom 文件中引入 cache 相关的依赖。

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```



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

容器启动的时候通过 aop 的方式为标注了 cache 相关注解的方法添加切点和方法，主要的类有以下几个：`ProxyCachingConfiguration` 配置 Cache AOP 的主要配置类。`BeanFactoryCacheOperationSourceAdvisor`  即 Advisor，`CacheInterceptor` 切点方法(Advice)，`CacheOperationSourcePointcut` 切点。

`ProxyCachingConfiguration` 主要代码如下：

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

  // 为容器中添加 Advisor, 其内部包含 PointCut
	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
		//....
	}

  // 为容器引入 CacheInterceptor
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor() {
		//...
	}
}

```



Advisor, Advice 和 PointCut 的关系可以使用下图表示，即 Advisor 包含 Advice 和 PointCut，PointCut 表明在什么地方执行方法，Advice 表示则表示在方法之前之后还是包含的方法的方式执行具体什么方法。

![Adivsor, Advice, PointCut](https://i.loli.net/2020/06/30/z3C5ErPK7ti4XU9.png)

 `CacheOperationSourcePointcut` 比较简单，其继承了 `StaticMethodMatcherPointcut`，拿到所有标注了cache 相关注解的方法。

`CacheInterceptor` 也比较简单，简单的看为一个 around advice，主要方法在其父类 `CacheAspectSupport` 实现，即 `execute` 该方法会在标注了 cache 相关注解的方法调用的时候执行。

#### 运行的步骤

如果一个方法标注了 `@Cacheable `  注解，那么每次在执行方法之前都会去检查缓存中是否已经存在了，如果对应的 cache 不存在则会根据配置的 cache name 创建一个 cache，然后根据生成的 key 从 cache 中找对应的 item，如果存在则直接返回 cache 中的值，否则会继续执行方法然后将方法返回的结果放到 cache 中。简单的调用图如下：

![简单的调用流程](https://i.loli.net/2020/06/30/UdPbFvSHWg4kxKB.png)

上图调用流程完毕之后会执行下面的代码：

```java
// Collect puts from any @Cacheable miss, if no cached item is found
List<CachePutRequest> cachePutRequests = new LinkedList<>();
if (cacheHit == null) {
  collectPutRequests(contexts.get(CacheableOperation.class),
                     CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
}

Object cacheValue;
Object returnValue;
if (cacheHit != null && !hasCachePut(contexts)) {
  // If there are no put requests, just use the cache hit
  cacheValue = cacheHit.get();
  returnValue = wrapCacheValue(method, cacheValue);
}
else {
  // Invoke the method if we don't have a cache hit
  returnValue = invokeOperation(invoker);
  cacheValue = unwrapReturnValue(returnValue);
}
// Collect any explicit @CachePuts
collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

// Process any collected put requests, either from @CachePut or a @Cacheable miss
for (CachePutRequest cachePutRequest : cachePutRequests) {
  cachePutRequest.apply(cacheValue);
}

// Process any late evictions
processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);
```

补充一点就是生成 key 的逻辑：

```java
// 如果定义了 SPEL 则根据 SPEL 生成，否则则直接使用 SimpleKeyGenerator
protected Object generateKey(@Nullable Object result) {
  if (StringUtils.hasText(this.metadata.operation.getKey())) {
    EvaluationContext evaluationContext = createEvaluationContext(result);
    return evaluator.key(this.metadata.operation.getKey(), this.metadata.methodKey, evaluationContext);
  }
  return this.metadata.keyGenerator.generate(this.target, this.metadata.method, this.args);
}

// 默认规则，如果没有参数则使用空 SimpleKey， 如果只有一个参数则直接使用该参数，如果有多个参数则包装成 SimpleKey
public static Object generateKey(Object... params) {
  if (params.length == 0) {
    return SimpleKey.EMPTY;
  }
  if (params.length == 1) {
    Object param = params[0];
    if (param != null && !param.getClass().isArray()) {
      return param;
    }
  }
  return new SimpleKey(params);
}
```

到这里缓存运行的部分就结束了。

### Cache 注解相关属性

首先看一下 `keyGenerator` 配置，如果默认的 `SimpleKeyGenerator` 无法满足需求的话也可以自定义一个 `KeyGenerator`，然后通过 `keyGenerator` 配置自定义的 Key Generator 例如：

```java
@Bean("customizedKeyGenerator")
public KeyGenerator keyGenerator() {
  return new KeyGenerator() {
    @Override
    public Object generate(Object target, Method method, Object... params) {
      return method.getName() + "::" + Arrays.asList(params).toString();
    }
  };
}
```

```java
@Cacheable(cacheNames="books", keyGenerator="customizedKeyGenerator")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

然后看一下 `condition` 的配置，例如只有在名称小于 32 的时候才进行缓存。

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32") 
public Book findBook(String name)
```

此外为了对 condition 进行补充，Spring 还提供了 `unless` 配置，即在上述的条件满足的情况下如果也满足 `unless` 中的条件则还是进行缓存的，例如：

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result.hardback") 
public Book findBook(String name)
```

当然也是支持 Optional 的，例如：

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result?.hardback")
public Optional<Book> findBook(String name)
```

默认情况下是以异步的方式操作缓存的，针对于多线程的情况，也可以指定缓存以同步的方式进行，只需要设置 `sync` 属性即可，如果设置了该属性那么则无法使用 `unless` 属性，也不能指定多个缓存。

```java
@Cacheable(cacheNames="foos", sync=true) 
public Foo executeExpensiveOperation(String id) {...}
```

针对于需要复杂的 cache 配置的情况可以使用 `@Caching` 注解，例如下面的例子

```java
@Caching(
  cacheable = {@Cacheable(value = "bk1", key = "#id")},
  put = {
    @CachePut(value = "bk2", key = "#result.name"),
    @CachePut(value = "bk2", key = "#result.title")
  },
  evict = {
    @CacheEvict(value = "list", allEntries = true)
  }
)
public Book updateBooking(String id) {...}
```

另外如果某些定制选项应用于类的所有操作，那么配置起来可能会很麻烦。例如，为类的每个缓存操作指定要使用的缓存名称可以由单个类级别定义替换，这里可以使用 `@CacheConfig` 配置，例如一个简单的配置如下：

```java
@CacheConfig("books") 
public class BookRepositoryImpl implements BookRepository {
    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```

### 使用 Redis 作为缓存

如果使用 Spring Boot 则可以通过 pom 文件引入 redis 相关的依赖，例如：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

简单的配置如下：

```yaml
spring:
  redis:
    port: 6002
    host: *.*.*.*
    #password: ***
    #database: *
```

如果引用了 Redis 相关的依赖哪儿 `RedisCacheConfiguration` 则会生效，为容器引入 `RedisCacheManager` 用于管理 Redis 的 Cache 相同的内容。当然和其他类似的依赖类似，也可以通过  去自定义 Redis 相关的内容，简单的例子如下：

```java
@Bean
public RedisCacheManagerBuilderCustomizer myRedisCacheManagerBuilderCustomizer() {
  return (builder) -> builder
    .withCacheConfiguration("cache1",
                            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofSeconds(10)))
    .withCacheConfiguration("cache2",
                            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(1)));

}
```

配置了以上的内容之后在此使用到缓存的时候就会自动使用 redis 作为缓存了。

在新版本的 Spring boot 中默认使用 `Lettuce` 作为默认的  Redis Client

```log
LettuceConnectionConfiguration#redisConnectionFactory matched:
      - @ConditionalOnMissingBean (types: org.springframework.data.redis.connection.RedisConnectionFactory; SearchStrategy: all) did not find any beans (OnBeanCondition)
```

#### Redis 自动配置过程

首先看一下 `RedisAutoConfiguration` 这个配置类，在引入了 Redis 相关的依赖的时候才会起作用，其为容器中引入了两份 Config 和两个 Bean，Config 分别是 `LettuceConnectionConfiguration` 和 `JedisConnectionConfiguration` 即两个 Redis Client 的客户端配置，两个 Bean 是操作 Redis 的 Template 是对 Redis 操作的封装（PS: 感觉 Spring 提供了各种各样的 Template）。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}

```

然后看一下 `LettuceConnectionConfiguration` Spring Boot 默认是使用这一份配置，其主要为容器引入了 `LettuceConnectionFactory` 实现 `RedisConnectionFactory `接口。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisClient.class)
class LettuceConnectionConfiguration extends RedisConnectionConfiguration {

	LettuceConnectionConfiguration(RedisProperties properties,
			ObjectProvider<RedisSentinelConfiguration> sentinelConfigurationProvider,
			ObjectProvider<RedisClusterConfiguration> clusterConfigurationProvider) {
		super(properties, sentinelConfigurationProvider, clusterConfigurationProvider);
	}

	@Bean(destroyMethod = "shutdown")
	@ConditionalOnMissingBean(ClientResources.class)
	DefaultClientResources lettuceClientResources() {
		return DefaultClientResources.create();
	}

	@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	LettuceConnectionFactory redisConnectionFactory(
			ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
			ClientResources clientResources) throws UnknownHostException {
		LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(builderCustomizers, clientResources,
				getProperties().getLettuce().getPool());
		return createLettuceConnectionFactory(clientConfig);
	}
	// 	...
}
```

然后看一下 Redis Cache 相关的配置, 主要是 `RedisCacheConfiguration` 配置类，其在 `RedisConnectionFactory` 存在的情况下才会起作用，所以需要引入 Redis 相关的依赖的，`RedisAutoConfiguration` 起作用后才会生效。其主要为容器引入了 `RedisCacheManager` 用于管理 Cache。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisConnectionFactory.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
@ConditionalOnBean(RedisConnectionFactory.class)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class RedisCacheConfiguration {

	@Bean
	RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
			RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
		RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
				determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
		List<String> cacheNames = cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
			builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
		}
		redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
		return cacheManagerCustomizers.customize(builder.build());
	}
	//...
}

```

有了 `RedisCacheManger`, 即可以通过其获取 `RedisCache` ，`RedisCache` 会根据缓存的操作处理 Redis 中的数据。对于 Redis 来说获取 Cache 和 从 Cache 中获取具体的 Item 流程如下，对于 `getCache` 来说会使用 `AbstractCacheManager` 中的 `getCache` 方法。

```java
public Cache getCache(String name) {
  // Quick check for existing cache...
  Cache cache = this.cacheMap.get(name);
  if (cache != null) {
    return cache;
  }

  // The provider may support on-demand cache creation...
  Cache missingCache = getMissingCache(name);
  if (missingCache != null) {
    // Fully synchronize now for missing cache registration
    synchronized (this.cacheMap) {
      cache = this.cacheMap.get(name);
      if (cache == null) {
        cache = decorateCache(missingCache);
        this.cacheMap.put(name, cache);
        updateCacheNames(name);
      }
    }
  }
  return cache;
}
```

如果没有 cache 则会通过 `getMissingCache` 生成一个 Cache, `RedisCacheManger` 中的实现如下：

```java
protected RedisCache getMissingCache(String name) {
  return allowInFlightCacheCreation ? createRedisCache(name, defaultCacheConfig) : null;
}
protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
  return new RedisCache(name, cacheWriter, cacheConfig != null ? cacheConfig : defaultCacheConfig);
}
```

在引入 `RedisCacheManger` 的时候就已经引入一份 默认的 cache 配置。而从 Cache 中查找缓存条目则在 `RedisCache` 中通过 `RedisCacheWriter` 去操作 Redis 数据库。

```java
protected Object lookup(Object key) {
  byte[] value = cacheWriter.get(name, createAndConvertCacheKey(key));
  if (value == null) {
    return null;
  }
  return deserializeCacheValue(value);
}
```

#### Redis 序列化机制

上面也提到了 Redis Auto Config 会为容器中引入两个 Template 分别是： `RedisTemplate` 和 `StringRedisTemplate` 。前者操作的是都是对象，后者则是操作 String。对于 `RedisTemplate` 来说默认使用的 JDK 序列化方式。

```java
if (defaultSerializer == null) {
  defaultSerializer = new JdkSerializationRedisSerializer(
    classLoader != null ? classLoader : this.getClass().getClassLoader());
}
```

可以替换 JSON 的序列化方式：

```java
@Bean
public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
  RedisTemplate<Object, Object> template = new RedisTemplate<>();
  template.setConnectionFactory(redisConnectionFactory);
  template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer(new ObjectMapper()));
  return template;
}
```

#### 自定义 `CacheManager`

首先看一下默认的 `cacheManager` 的实现如下, 默认使用的是 JDK 的序列化机制，如果要替换成 JSON 的序列化方式可以通过自定义一个 CacheManager 的方式。

```java
@Bean
RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers,
                               ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
                               ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
                               RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
  RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
    determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
  List<String> cacheNames = cacheProperties.getCacheNames();
  if (!cacheNames.isEmpty()) {
    builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
  }
  redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
  return cacheManagerCustomizers.customize(builder.build());
}

private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
  CacheProperties cacheProperties,
  ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
  ClassLoader classLoader) {
  return redisCacheConfiguration.getIfAvailable(() -> createConfiguration(cacheProperties, classLoader));
}

private org.springframework.data.redis.cache.RedisCacheConfiguration createConfiguration(
  CacheProperties cacheProperties, ClassLoader classLoader) {
  Redis redisProperties = cacheProperties.getRedis();
  org.springframework.data.redis.cache.RedisCacheConfiguration config = org.springframework.data.redis.cache.RedisCacheConfiguration
    .defaultCacheConfig();
  config = config.serializeValuesWith(
    SerializationPair.fromSerializer(new JdkSerializationRedisSerializer(classLoader)));
  //...
  return config;
}
```

例如自定义一个 `CacheManager`， 简单的示例如下：

```java
@Bean
public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
   return RedisCacheManager.builder(RedisCacheWriter.lockingRedisCacheWriter(connectionFactory))
     .cacheDefaults(defaultCacheConfig(new ObjectMapper()))
     .withInitialCacheConfigurations(singletonMap("demo", defaultCacheConfig(new ObjectMapper())))
     .transactionAware()
     .build();
 }

@Bean
@Primary
public RedisCacheConfiguration defaultCacheConfig(ObjectMapper objectMapper) {
  return RedisCacheConfiguration.defaultCacheConfig()
    .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
    .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
}
```

当然也可以通过设置 `RedisCacheManagerBuilderCustomizer` 的方式设置 JSON 的序列化方式：

```java
@Bean
public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
  return builder -> builder.cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                                          .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                                          .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())));
}
```

