## Dubbo 基本配置

参考地址：[Dubbo 配置](https://dubbo.apache.org/zh-cn/docs/user/demos/preflight-check.html)

**启动时检查**

Dubbo 默认情况下在启动的时候会检查依赖的服务是否可用，如果不可用的时候则抛出异常阻止 Spring 初始化的完成。如果是测试或者有循环依赖则需要关闭启动时检查，配置如下：

```java
// # 关闭某个服务的启动时没有提供者的 Check
@Reference(check = false)
ArticleService articleService;
```

```properties
# 关闭所有服务的启动时没有提供者的 Check
dubbo.consumer.check=false
dubbo.reference.check=false
# 关闭注册中心启动时注册订阅失败时的 Check，
dubbo.registry.check=false
```

**集群容错**

当集群调用失败时，Dubbo 提供了多种容错方案，默认是 `failover` 重试。可选的容错策略有以下几种：

1. `failover` 失败自动切换，可以通过设置 `retries` 指定重试次数（不含第一次），默认是 2 次。
2. `failfast` 快速失败，失败立即报错，通常用于非幂等性操作，例如添加记录。
3. `failsafe` 失败安全，出现异常时忽略，通常用于写入审计日志等。
4. `failback` 失败自动回复，后台记录失败请求，定时重发。通常用于消息通知操作。
5. `forking` 并行调用多个服务，只要一个成功返回即返回，通常用于实时性要求较高的读操作，可以通过设置 `forks` 属性设置最大并行数量。
6. `broadcast` 广播调用者，逐个调用任意一台报错则报错，通常用于通知所有提供者更新缓存或者日志等本地资源信息。

一个简单的配置如下：

```java
@Reference(version = "1.0.1", cluster = "failsafe")
ArticleService articleService;
```

**负载均衡**

在集群负载均衡的时候 Dubbo 提供了多种策略，默认是 `random` 随机调用。主要有以下几种：

1. `random` 随机，按照权重设置随机率。
2. `roundrobin` 轮询，按照公约权重设置轮询比率。
3. `leastactive` 最小活跃调用数，相同活跃数的随机。使慢的提供者收到更少请求。
4. `consistenthash` 一致性 hash，相同参数的请求总是发到同一服务器

```java
@Reference(loadbalance = "roundrobin")
ArticleService articleService;
```

**直连消费者**

在开发及测试环境中，经常会绕过注册中心，只点对点直连测试服务提供者，其以服务接口为单位，忽略注册中心的提供者列表。配置如下：

```java
@Reference(url = "dubbo://localhost:20890")
ArticleService articleService;
```

**仅订阅及仅注册**

一个正在开发中的服务提供者可能会影响消费者不能正常运行，可以让服务提供者仅订阅服务（可能依赖其他服务）而不注册服务，通过直连的方式测试服务。

![仅订阅](http://img.programya.com/20200104000911.png)

与之对应的是值注册，例如两个注册中心，有一个服务职能在其中一个注册中心有部署，另一个注册中心还没有来得及部署，而两个注册中心的其他应用都依赖此服务。这个时候可以让服务提供者值注册服务到另一注册中心，而不从另一注册中心订阅服务。配置如下：

```yaml
dubbo:
  application:
    name: dubbo-provider
  registry:
    address: zookeeper://192.168.0.111:2181
    # 仅订阅
    register: false
    # 仅注册
    # subscribe: false
```

**静态服务**

如果需要人工管理服务提供者上下线，则可以将注册中心标识为非动态管理的模式：

```yaml
dubbo:
  application:
    name: dubbo-provider
  registry:
    address: zookeeper://192.168.0.111:2181
    dynamic: false
```

**服务版本**

当一个借口实现出现不兼容升级的时候可以使用版本号过度，版本号同步的服务间不引用，配置如下：

```java
// 设置提供者的版本号
@Service(version = "1.0.1")
public class ArticleServiceImpl implements ArticleService {}

// 使用的时候设置版本号
@Reference(version = "1.0.1")
ArticleService articleService;
```

**结果缓存**

结果缓存用于加速热门数据的访问速度，Dubbo 提供声明式缓存。主要有以下几种缓存类型：

1. `lru` 基于最少使用原则删除多余的缓存。
2. `threadlocal` 当前线程缓存
3. `jcache` 与 JSR 107 集成可以桥接各种缓存的实现。

配置如下：

```java
@Reference(cache = "lru")
ArticleService articleService;
```

**上下文信息**

上下文中存放的时当前调用过程中所需的环境信息，所有信息都转换成 URL 的参数，其中 `RpcContext` 是 `ThreadLocal` 的临时状态记录器，当接收到 RPC 请求或者发起 RPC 请求时，`RpcContext` 的状态会变化。一个简单的使用样例如下：

```java
// 本端是否为提供端，这里会返回true
boolean isProviderSide = RpcContext.getContext().isProviderSide();
// 获取调用方IP地址
String clientIP = RpcContext.getContext().getRemoteHost();
// 获取当前服务配置信息，所有配置信息都将转换为URL的参数
String application = RpcContext.getContext().getUrl().getParameter("application");
// 注意：每发起RPC调用，上下文状态会变化
yyyService.yyy();
// 此时本端变成消费端，这里会返回false
boolean isProviderSide = RpcContext.getContext().isProviderSide();
```

**Provider 和 Comsumer 异步调用**

**参数回调**

**本地存根**



**延迟暴露**

如果你的服务需要预热时间，比如初始化缓存等待相关资源等，可以使用 `delay` 进行延迟暴露。服务倒计时是从 Spring 初始化完成后进行的。

```java
@Service(delay = 1000)
public class ArticleServiceImpl implements ArticleService {}
```

**并发控制与连接控制**

```java
// 并发控制
@Service(executes = 10)
public class ArticleServiceImpl implements ArticleService {}

// 连接控制
@Service(connections = 10)
public class ArticleServiceImpl implements ArticleService {}
```

**路由规则**

**服务降级**

可以通过服务降级功能临时屏蔽掉某个出错的非关键服务，并定义返回策略。可以向注册中心写入动态配置覆盖规则如下所示：

```java
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null"));
```

`mock=force:return+null` 表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。

还可以改为 `mock=fail:return+null` 表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。

