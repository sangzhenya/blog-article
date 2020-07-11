---
title: "Spring Boot 配置文件"
tags: ["Spring", "Spring Boot"]
categories: ["Spring"]
date: "2019-02-19T09:00:00+08:00"
---

Spring Boot 默认使用 properties 文件来配置属性，也可以使用 YAML 文件进行配置。YAML 以数据为中心，比 json 和 xml 等更适合做配置文件。

**基本语法**

K:**[空格]**V	表示一对键值对。以**空格**的缩进控制层级关系, 属性和值均是大小写敏感的，只要是左对齐的一列数据，都是同一个层级的。

```yaml
server: 
    port: 8080
```

**普通值**

1. 数字
2. 字符串
3. 布尔

字符串默认不用加上单引号或者双引号。`""` 双引号不会转义字符串里面的特殊字符；`''`单引号则会转义特殊字符，特殊字符最终只是一个普通的字符串数据。例如：

```yaml
name: "xinyue \n lin"

# 打印输出为
# xinyue
# lin

name: 'xinyue \n lin'
# 打印输出
# xinyue \n lin
```

**Map  属性值**

可以使用多行表示也可以使用行内表示，例如

```yaml
# 多行表示
person: 
    firstName: xinyue
    lastName: lin
    
# 行内表示
person: {firstName: xinyue, lastName: lin}
```

**List/Set 数组**

可以使用 `-` 多行表示，也可以使用行内表示，例如

```java
phones: 
    - apple
    - xiaomi
    - rongyao
    
phones: [apple, xiaomi, rongyao]
```



**配置文件注入**

配置文件中的配置可以映射到 Java 的一个 Bean。使用 `@ConfigurationProperties` 支持松散绑定（驼峰命名，横线拼接等）， JSR303 数据校验，复杂类型封装(例如 Map/List)。但是不支持 SpEL 语法。可以使用 `@PropertySource` 和 `@ImportResource`

```java
@PropertySource(value = {"classpath:person.yaml"})
@ImportResource(locations = {"classpath:person.yaml"})
```

Java Bean:

```java
@Component
@ConfigurationProperties(prefix = "person")
public class PersonConfig {
    private int id;
    private String name;
    @Email	// 邮箱校验
    private String email;
    private String addr;
    private int age;
    private List<String> phones;
    private Map<String, String> propertyMap;
    /*Setter...Getter*/   
}
```

YAML 文件：

```java
person:
  id: 101
  name: Xinyue
  email: xinyue@demo.com
  addr: My Addr
  age: 20
  phones:
    - xiaomi
    - apple
  property-map:
    pro1: value1
    pro2: value2
    pro3: value3
```

可以导入 `configuration-processor` 包，之后配置的时候便会有提示了，pom 文件如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```



使用 `@Value` 获取值， 参考： [Spring 注解 --- 初始化、销毁及赋值](https://programya.com/html/article/102)



**配置文件可以使用占位符**

1. 随机数

   `${random.value}`, `${random.uuid}`, `${random.int}`, `${random.int(10)}`, `${random.int[1, 100]}`, `${random.long}`

2. 获取之前配置的值（可以指定默认值）

   ```java
   person:
     id: 101
     name: Xinyue
     mark-name: mark ${person.name:默认值}
   ```



**多配置文件**

Spring Boot 默认使用： `application.properties` 文件，指定多配置文件的时候可以使用 `application-{profile}.properties/yaml` 的命名方式

YAML 本身也支持多文档块的形式，例如：

```yaml
server: 
    port: 8079
spring: 
    profiles: 
        active: dev

---
server: 
    port: 8079
spring: 
    profiles: dev
    
---
server: 
    port: 8079
spring: 
    profiles: prd
```



激活指定 Profile 有以下几种方式：

1. 配置文件中指定：`spring.profiles.active=dev`
2. 命令行指定： `java -jar boot.jar --spring.profile.active=dev`
3. 虚拟机参数：`-Dspring.profiles.active=dev`



**配置文件会从以下位置加载**

Spring Boot 启动会扫描以下位置的 `application.properties` 或者 `application.yaml` 作为 Spring Boot 的默认配置文件，优先级从高到低，高优先级配置覆盖低优先级覆盖，进行互补配置。

1. file:./config/
2. file:./
3. classpath:/config/
4. classpath:/

可以使用 `spring.config.location` 改变默认的配置文件位置。在启动项目的时候通过命令行参数的形式指定配置文件：`java -jar boot.jar --spring.config.location=/home/xinyue/application.properties`。



**配置加载顺序**

1. 命令行参数，多个配置使用空格分开

   `java -jar boot.jar --server.port=8081 --service.context-path=/home`

2. 来自 `java:comp/env` JNDI 属性

3. Java 系统属性，可以使用 `System.getProperties()` 查看具体可用属性

4. 系统系统环境变量

5. RandomValuePropertySource 配置的 rand.* 属性值

6. 包外 `spring-{profile}.properties/yaml`

7. 包内 `spring-{profile}.properties/yaml`

8. 包外 `spring.properties/yaml`

9. 包内 `spring.properties/yaml`

10. `@Configuration` 注解类上的 `@PropertySource`

11. 使用代码 `SpringApplication.setDefaultProoperties` 指定的默认属性



