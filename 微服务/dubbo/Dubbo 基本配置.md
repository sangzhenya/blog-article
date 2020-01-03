## Dubbo 基本配置

参考地址：[Dubbo 配置](https://dubbo.apache.org/zh-cn/docs/user/demos/preflight-check.html)

启动时检查

Dubbo 默认情况下在启动的时候会检查依赖的服务是否可用，如果不可用的时候则抛出异常阻止 Spring 初始化的完成。如果是测试或者有循环依赖则需要关闭启动时检查，配置如下：

```properties
# 关闭某个服务的启动时没有提供者的 Check
dubbo.reference.com.foo.BarService.check=false
dubbo.reference.check=false
# 关闭所有服务的启动时没有提供者的 Check
dubbo.consumer.check=false
# 关闭注册中心启动时注册订阅失败时的 Check，
dubbo.registry.check=false
```



1. 集群容错
2. 负载均衡
3. 直连消费者
4. 仅订阅及仅注册
5. 静态服务
6. 服务版本
7. 结果缓存
8. 上下文信息
9. Provider 和 Comsumer 异步调用
10. 参数回调
11. 本地存根
12. 延迟暴露
13. 并发控制与连接控制
14. 路由规则
15. 服务降级

