## Actuator

通过引入 `spring-boot-starter-actuator` 可以使用 Spring Boot 为我们提供的应用的监控和管理功能。可以通过 HTTP，JMX，SSH 协议进行操作，自动得到审计、健康及指标信息。pom 文件中设置如下：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

主要有以下一些内容：

1. `/actuator` 启用了哪些 `actuator`

2. `/actuator/beans` 容器中每个组件的定义信息

3. `/actuator/caches` 容器中的 Cache 信息

4. `/actuator/caches/{cache}` 容器中具体 Cache 的缓存条目

5. `/actuator/health/{*path}` 和 `/actuator/health` 应用的健康状况

6. `/actuator/info` 应用自定义显示的信息

   可以通过下面 的样式设置

   ```yaml
   info:
     application:
       id: 9573
       name: DEMO
   ```

   也可以添加 `git.properties` 配置

   ```properties
   git.commit.time=2020-04-04 12:12:12
   git.commit.id=iikj
   git.branch=master
   ```

   

7. `/actuator/conditions` 自动配置报告

8. `/actuator/shutdown` 关闭当前应用，默认是关闭的需要通过配置启用

9. `/actuator/configprops`  系统 actuator 配置属性

10. `/actuator/env/{toMatch}` 和 `/actuator/env` 系统环境信息

11. `/actuator/loggers/{name}` 和 `/actuator/loggers`  系统 loggers 信息d

12. `/actuator/heapdump` dump 一份应用的 JVM 堆信息

13. `/actuator/threaddump` 线程活动的快照

14. `/actuator/metrics/{requiredMetricName}` 和 `/actuator/metrics` 系统度量信息

15. `/actuator/scheduledtasks` 系统中定时任务的信息

16. `/actuator/mappings` url 映射关系

以上的 URL 中默认启用的只用： `/actuator/health` 和 `/actuator/info` 两个，可以通过以下的配置启用全部或者部分，也可以使用 `exclude` 属性指定排除的元素。也可以直接配置某些 endpoint 启用，此外这些 endpoint 的地址也可以自定义，例如自定义 `beans` 的 Path 为 `ibean` ，定义 base url 为 `/health`

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
        exclude: 'mappings'
      path-mapping:
        beans: ibean
      base-path: /health
    threaddump:
    	enabled: false
```

也可以自定义一个 health indicator 如下：

```java
@Component
public class MyHealthIndicator extends AbstractHealthIndicator {
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        builder.up().status(Status.UP).withDetail("Up Time", new Date());
    }
}
```

此外需要配置 health show details:

```yaml
management:
  endpoint:
    health:
      show-details: always
```





