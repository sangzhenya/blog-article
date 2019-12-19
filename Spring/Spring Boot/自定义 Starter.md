## 自定义 Starter

Starter 只是用来引入自动配置模块，Starter 依赖自动配置，其他人使用的时候只需要引入 Starter 即可。另外对于我们自己实现的 Starter 推荐命名是 `xxx-spring-boot-starter`。

首先创建一个空项目例如 `mystarter`, 打开项目创建两个 Module，一个是普通的 Maven 项目，一个是 Spring Boot 项目。如下图所示：

![Module 图](http://img.sangzhenya.com/Snipaste_2019-12-19_23-55-27.png)

对于 Starter 项目非常简单，只需要在 `pom.xml` 文件中引入 `autoconfig` 项目依赖即可。

```xml
<dependencies>
  <dependency>
    <groupId>com.xinyue</groupId>
    <artifactId>xinyue-spring-boot-starter-autoconfig</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </dependency>
</dependencies>
```

然后编写 `autoconfig` 项目，其是一个 Spring Boot 项目所以依赖于 SpringBoot。

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
```

由于是被其他人使用的第三方模块，所以不需要主类，如果自动生成了主类可以删除。然后就可以编写代码。

首先编写一个 Service 即给外部用的业务方法，简单如下：

```java
public class StarterService {
    private final StarterProperties starterProperties;
    public StarterService(StarterProperties starterProperties) {
        this.starterProperties = starterProperties;
    }
    public String hello(String name) {
        return starterProperties.getPrefix() + name + starterProperties.getSuffix();
    }
}
```

如果使用到了配置，例如上面使用到了 `StarterProperties`，则可以自定义配置，一个简单的 `properties` 如下：

```java
@ConfigurationProperties("xinyue.demo")
public class StarterProperties {
    private String prefix;
    private String suffix;
    public String getPrefix() {
        return prefix;
    }
    public void setPrefix(String prefix) {
        this.prefix = prefix;
    }
    public String getSuffix() {
        return suffix;
    }
    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }
}
```

最后配置 AutoConfig 类，将 Bean 加入到容器中，如下所示：

```java
@Configuration
@ConditionalOnWebApplication
@EnableConfigurationProperties(StarterProperties.class)
public class StarterAutoConfig {
    private final StarterProperties starterProperties;
    public StarterAutoConfig(StarterProperties starterProperties) {
        this.starterProperties = starterProperties;
    }
    @Bean
    public StarterService starterService() {
        return new StarterService(starterProperties);
    }
}
```

最后在 `resource` 文件夹新建 `META-INF` 文件夹，在其中新建 `spring.factories` 文件，写入 AutoConfig 类的全路径，如下哦所示：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.xinyue.config.StarterAutoConfig
```



到这里一个简单地  Spring Boot Starter 已经编写完毕了，只要依次将 `starter` 和 `autoconfig` 项目 install 到 Maven 仓库中即可。

在使用的地方引入 Starter 的依赖即可，例如：

```xml
<dependency>
  <groupId>org.xinyue</groupId>
  <artifactId>xinyue-starter</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>
```

然后再项目的配置文件中就可以使用 自定义 Starter 的配置了，例如：

```yaml
xinyue:
  demo:
    prefix: hello,
    suffix: .lalalala
```

也可以调用自定义 Starter 的业务方法了，例如:

```java
private StarterService starterService;
public DemoController(StarterService starterService) {
  this.starterService = starterService;
}

@ResponseBody
@GetMapping("starter")
public String starter() {
  return starterService.hello("starter");
}
```

最后启动服务，调用结果如下:

![结果](http://img.sangzhenya.com/Snipaste_2019-12-20_00-08-38.png)

