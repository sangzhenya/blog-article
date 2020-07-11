---
title: "Dubbo 基本概念和环境搭建"
tags: ["Dubbo"]
categories: ["微服务"]
date: "2020-02-01T09:00:00+08:00"
---

参考： [Dubbo  快速入门]( https://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

常规的垂直应用架构已无法应对网站应用规模的不断扩大，互联网应用架构大致的演进过程如下：

![互联网应用演进过程](http://img.programya.com/20191231174104.png)

大致的演进历程也就是： 单一应用架构 -> 垂直应用架构 -> 分布式服务架构 -> 流式计算架构。

在流式计算架构中有一个调度中心基于访问压力实时管理集群容量，提高集群利用率。

Dubbo 是一款高性能Java RPC框架。RPC 即 Remote Procedure Call，远程过程调用，是一种进程间通信方式，其实一种技术思想。运行程序调用另外一个地址空间的欧过程或函数，而不用显示编码这个远程调用的细节。RPC 框架的基本原理图如下：

![RPC 基本原理图](http://img.programya.com/20191231181843.png)

主要是以下几步：

1. Client 像调用本地服务似的调用远程服务
2. Client stub 接收到调用后，将方法、参数序列化
3. 客户端通过 socket 将消息发送服务端
4. Server stub 收到消息后进行解码，即将消息对象反序列化
5. Server stub 根据解码结果调用本地的服务
6. 本地服务执行并将结果返回给 Server stub
7. Server stub 将返回结果打包成消息
8. 服务端通过 Sockets 将消息发送到客户端
9. Client stub 接收到结果消息并进行解码，即将消息反序列化
10. 客户端得到最终结果

细化下来的 RPC 各个组成部分如下：

![RPC 主要组成部分](http://img.programya.com/20191231181610.png)

Dubbo 主要有以下特性：连通性，健壮性，伸缩性以及面向未来架构的伸缩性。Dubbo 的基础架构如下图所示：

![Dubbo 基础架构](http://img.programya.com/20191231183353.png)

其中 Provider 是暴露服务的服务提供方，Consumer 是调用远程服务的服务消费方，Registry 是服务注册与发现中心，Monitor 是统计服务的调用次数和调用时间监督中心，Container 是服务运行容器。上图的流程大致如下：

0. 服务容器负责启动，加载，运行服务提供者
1. 服务提供者在启动时向注册中心注册自己提供的服务
2. 服务消费者在启动时向注册中心注册自己所需的服务
3. 注册中心返回服务提供者地址列表给消费者，如果有变更注册中心将基于长连接推送变更数据给消费者。
4. 服务消费者从提供者地址列表中基于软负载均衡算法选一天提供者调用，如果调用失败再选择另一台。
5. 服务消费者和提供者在内存中累计调用的次数合适间定时每分钟发送一次统计数据到监控中心。



Dubbo 可以使用 JVM 参数配置，XML 配置以及 属性配置等三种配置方式设置相关的参数。优先级顺序如下图所示：

![Dubbo 配置属性优先级](http://img.programya.com/20200102214640.png)

一个简单的 Spring Boot 整合 Dubbo 项目地址：[GitHub 地址](https://github.com/sangzhenya/code-samples/tree/master/dubbo)

主要的依赖如下：

```xml
<dependencies>
  <!-- https://mvnrepository.com/artifact/org.apache.dubbo/dubbo-spring-boot-starter -->
  <dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.4.1</version>
  </dependency>

  <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>

  <!-- https://mvnrepository.com/artifact/org.apache.curator/curator-framework -->
  <dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.2.0</version>
  </dependency>

  <!-- https://mvnrepository.com/artifact/org.apache.curator/curator-recipes -->
  <dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.2.0</version>
  </dependency>

</dependencies>
```

Provider 的 Service 实现如下：

```java
@Service(version = "1.0.1")
public class ArticleServiceImpl implements ArticleService {
    @Override
    public Article getArticleById(Integer id) {
        Article article = new Article();
        article.setId(id);
        article.setTitle("Title: " + id);
        article.setSummary("Summary: " + id);
        article.setCreateDate(LocalDateTime.now());
        article.setLastUpdateDate(LocalDateTime.now());
        return article;
    }
}
```

配置如下：

```yaml
server:
  port: 8080
dubbo:
  application:
    name: dubbo-provider
  registry:
    address: zookeeper://192.168.0.111:2181
  protocol:
    name: dubbo
    port: 20080
  scan:
    base-packages: com.xinyue.dubbo.provider.service
  metadata-report:
    address: zookeeper://192.168.0.111:2181
  provider:
    group: blog

```

Consumber 使用如下：

```java
@Reference(version = "1.0.1")
ArticleService articleService;

@GetMapping("/")
public Article hello() {
    return articleService.getArticleById(1);
}
```

配置如下：

```yaml
server:
  port: 8080
dubbo:
  application:
    name: dubbo-consumber
  registry:
    address: zookeeper://192.168.0.111:2181
  protocol:
    name: dubbo
    port: 20081
  scan:
    base-packages: com.xinyue.dubbo.provider.service
  metadata-report:
    address: zookeeper://192.168.0.111:2181
  consumer:
    group: blog
```

