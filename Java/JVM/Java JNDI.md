---
title: "Java JNDI"
tags: ["Java", "JVM"]
categories: ["Java"]
date: "2020-03-11T09:00:00+08:00"
---

JNDI 即 Java Naming and Direcotry Interface 即 Java 命名与目录接口。是为 Java 编程的客户端提供的一个统一的命名服务和目录系统接口，不依赖于任何特定的服务实现。JDNI 的架构包含一个 API和一个 service provider interface(SPI)。Java 程序使用 JNDI API 去访问不同的命名服务和目录系统接口。SPI 则使得不同的命名服务和目录系统的能够透明的镶嵌进来，进而使得 Java 程序可以通过这些 JNDI API 访问不同的服务。架构如下：

![JNDI 架构](https://i.loli.net/2019/08/28/LvnRHXyStFm8PY7.png)

如果要使用 JNDI，首先要有 JNDI 的类并且有至少一个 servier provider, JDK 中包含了下面一些命名服务和目录系统的 service:

1. Lightweight Directory Access Protocol (LDAP)
2. Common Object Request Broker Architecture (CORBA) Common Object Services (COS) name service
3. Java Remote Method Invocation (RMI) Registry
4. Domain Name Service (DNS)



### Naming 包

`javax.name` 包中包含了一系列访问命名服务的类和接口。其中 `Context` 是一个核心的接口，用于 looking up, binding/unbindiing, renaming objects 和 creating and destorying subcontexts.

+ Lookup, 最常使用的方法 `lookup()` ，可以根据提供的名称返回绑定在这个名称的上的 Object。

+ Binding, `listBindings()` 返回一个 name-to-object 绑定的集合，一个 binding 是一个包含了名称和绑定对象的元祖，`list()` 和 `listBindings()` 类似，只是其提供的是所有名名称的集合，对于只想获取绑定了哪些名称，而不是想要真正的获取这些对象而言是很合适的。

+ Name, 是一个泛型名称接口，零到多个组件的有序序列。

+ References，对象在命名服务与目录系统中存储方法可能各种各样，而 reference 是对象紧凑的一种表现形式， JNDI 定义了 Reference 类去表示 reference。一个 reference 包含了去创建一个对象的所需要的信息，

在  JNDI 中，所有的命名服务和目录系统的都是相对于上下文执行的，所以 JNDI 定义了 `InitialContext` 作为一个命名服务和目录系统的起始点，只有在此之后才可以使用去 look up 其他 context 和  objects。

简单来讲命名服务就是值与值之间的映射或者说是一组数据的别名，一组计算机容易认识的值的别名。

### JNDI 的使用的一个例子

访问 MySQL 数据库的时候首先要要根据驱动类名称获取 MySQL 的 JDBC 驱动，然后使用 JDBC URL 连接到数据库，才能继续进行数据操作：

```java
// 获取驱动
Class.forName("com.mysql.jdbc.Driver", true, Thread.currentThread().getContextClassLoader()); 
// 建立
conn=DriverManager.getConnection("jdbc:mysql://localhost:3306/demo?user=demo&password=demo 
```

考虑到数据库驱动，服务器，用户名和口令以及一些其他配置都可能更改，所以显然这样硬编码是不好的。有了 JNDI 之后就可以在 J2EE 中容器中配置 JNDI 参数，定义一个数据源并设置名称，程序中通过数据源名称引用数据源进而访问数据库。

例如在 Tomcat 中配置 JNDI 数据源，是在 tomcat 的 `/conf/context.xml` 文件中配置 JNDI  数据源，程序便可以通过名称找到对应的数据源对象。配置如下：

```xml
<!-- JDBC数据源 -->
<Resource name="jndi-demo"
  auth="Container" 
  type="javax.sql.DataSource"
  driverClassName="com.mysql.jdbc.Driver"
  url="jdbc:mysql://localhost:3306/demo"
  username="demo"
  password="demo"
  maxActiv ="20"
  maxIdle="1"
  maxWait="1"
/>
```

上述文件定义了一个名称叫做 `jndi-demo`  的数据源，当然可以定义多个 Resource 去连接不同的数据库。

`web.xml` 中需要如下配置：

```xml
<resource-ref>
  <description>jndi demo</description>
  <res-ref-name>jndi-demo</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
  <res-auth>Container</res-auth>
</resource-ref>
```

程序中则直接通过下面的方式引用：

```java
Context context = new InitialContext();
DataSource dataSource = (DataSource)context.lookup("java:comp/env/jndi-demo");
Connection Connection conn = dataSource.getConnection();
```

*`java:comp/env` 是一个 JNDI 的节点，在这个节点上你可以找到当前 J2EE 容器组件的配置信息*

如果使用了 Spring 容器，则可以在 配置文件中 dataSource Bean 中进行如下的配置：

```xml
<bean id="dataSource" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName">
        <value>java:comp/env/jndi-demo</value>
    </property>
</bean>
```

如果使用的是 SpringBoot 则在配置中对 dataSource 的配置如下：

```properties
spring.datasource.jndi-name=java:comp/env/jndi-demo
```



参考：

1. [JNDI是什么](https://jiges.github.io/2017/12/08/JNDI是什么/)