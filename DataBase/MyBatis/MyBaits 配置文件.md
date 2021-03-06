---
title: "MyBatis 配置文件"
tags: ["MyBaits"]
categories: ["MyBaits"]
date: "2019-05-09T09:00:00+08:00"
---

MyBaits 是一个半自动化的持久化层框架，支持定制化 SQL，存储过程以及高级映射。避免 JDBC 代码和手动设置参数以及获取结果集。其使用简单的 XML 或注解用于配置和原始映射，将接口和 Java 的对象映射成数据库记录。

![MyBaits 处理流程](http://img.programya.com/20191224220407.png)

一个简单的项目配置如下：

`pom.xml` 配置如下：

```xml
<dependencies>
  <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
  <dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.3</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.18</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/org.junit.jupiter/junit-jupiter-api -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.6.0-M1</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

首先创建一个 User 对象：

```java
public class User {
    private Integer id;
    private String name;
    private String password;
    private String lastName;
    private String firstName;
    // ... getter/setter
    // toString
}
```

然后创建一个 Mapper 接口

```java
public interface UserMapper {
    User getUserById(Integer id);
}
```

在 resource 目录下添加 DB 配置文件 `db.properties`

```properties
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/xinyue
jdbc.username=root
jdbc.password=123456
```

在 resource 目录下添加配置文件 `UserMapper.xml` 

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.xinyue.imybatis.mapper.UserMapper">
  <!-- namespace:名称空间;指定为接口的全类名 -->

  <!-- 对应方法是：User getUserById(Integer id); -->
  <!-- id：唯一标识; resultType：返回值类型; #{id}：从传递过来的参数中取出id值; databaseId 数据库厂商标识 -->
  <select id="getUserById" resultType="User"  databaseId="mysql">
		select id, name, password, first_name firstName, last_name lastName from user where id = #{id}
	</select>
</mapper>
```

在 resource 目录下添加配置文件 `mybatis-config.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 使用 properties 引入外部的配置文件 -->
  	<!-- 如果多个配置属性的地方优先级从高到低依次是：1. 方法参数；2. 通过 url 指定路径读取的文件内的属性；3. properties 元素体内指定的属性 -->
	  <properties resource="db.properties" />
    <!-- 重要的设置信息 -->
    <!-- 几个重要的配置如下：
 				1. cacheEnabled 缓存开关，默认为 true
 				2. lazyLoadingEnabled 延迟加载，默认为 false
				3. useColumnLabel 使用标签代替列名，默认为 true
				4. defaultStatementTimeout 超时时间秒数，默认为空
				5. mapUnderscoreToCamelCase 驼峰命名规则映射，默认为 false -->
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
  	<!-- 设置类的别称 -->
    <!-- 也可以使用 <package name="com.xinyue.imybatis.model"/> 指定一个包，然后在对应的包中实体类可以使用 @Alias("User") 注解设置别称 -->
    <typeAliases>
        <typeAlias type="com.xinyue.imybatis.model.User" alias="User" />
    </typeAliases>
  	<!-- MyBaits 可以配置多种环境，例如 Dev，QA 和 PRD，每个环境使用一个 environment 配置并指定唯一的标识符，可以通过 default 属性快速切换环境-->
    <environments default="development">
        <!-- id 表示当前环境的唯一标识 -->
        <environment id="development">
            <!-- 配置 JDBC TransactionManager -->
            <!-- JDBC: 使用了 JDBC 的提交和事务回滚设置，依赖于数据源得到的连接来管理事务范围 JdbcTransactionFactory -->
            <!-- JDBC: 使用了 MANAGED 不提交或回滚连接，让容器来管理事务的整个生命周期 ManagedTransactionFactory -->
            <!-- 自定义的全类名，自己实现 TransactionFactory 接口 -->
            <transactionManager type="JDBC" />
            <!-- 配置 Data Source -->
            <!-- UNPOOLED: 不使用连接池: UnpooledDataSourceFactory -->
            <!-- POOLED: 使用连接池: PooledDataSourceFactory -->
          	<!-- JNDI: 在 EJB 或应用应用服务中查找指定的数据源 -->
          	<!-- 自定义：实现 DataSourceFactory 接口，定义数据源获取方式 -->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}" />
                <property name="url" value="${jdbc.url}" />
                <property name="username" value="${jdbc.username}" />
                <property name="password" value="${jdbc.password}" />
            </dataSource>
        </environment>
    </environments>
  	
    <!-- 根据不同的数据库厂商执行不同的语句 -->
    <!-- type: DB_VENDOR, 使用 MyBatis 提供的 VendorDatabaseIdProvider 解析数据库厂商的标识。也可以实现 DatabaseIdProvider 接口来自定义 -->
    <!-- property name 数据库厂商标识 -->
    <!-- property value 数据库厂商标识别名，方便 SQL 语句使用 databaeId 属性引用 -->
  	<!-- 如果没有配置该标签 databaseIdProvider 那么 databaseId=null。如果配置了则使用标签的配置去匹配数据库的信息，如果匹配到则databaseId为指定的值，否则依旧为 null -->
  	<!-- 如果 databaseId 不为 null，则只会找配置 databaseId 的 SQL 语句。MyBatis 会加载不带 databaseId 属性和匹配当前数据库 databaseId 的所有语句，如果同时找到带有和不带有的则使用带有 databaseId 属性的 -->
    <databaseIdProvider type="DB_VENDOR">
        <property name="MySQL" value="mysql"/>
        <property name="Oracle" value="oracle"/>
        <property name="SQL Server" value="sqlserver"/>
     </databaseIdProvider>
  
  	<!-- 所有的 Mapper 放在下面 -->
  	<!-- 也可以使用 class 类 如下 -->
    <!-- <mapper class="com.xinyue.imybatis.mapper.UserMapper" /> 对应接口的方法上需要加上注解 例如 @Select("select id, name, password, first_name firstName, last_name lastName from user where id = #{id}") -->
    <!-- 也可以使用 <package name="com.xinyue.imybatis.model"/> 注册包中所有的，需要将 Mapper 文件和 接口类放在同目录下 -->
    <mappers>
        <mapper resource="UserMapper.xml" />
    </mappers>
</configuration>
```

测试方法如下：

```java
public SqlSessionFactory getSqlSessionFactory() throws IOException {
  String resource = "mybatis-config.xml";
  InputStream inputStream = Resources.getResourceAsStream(resource);
  return new SqlSessionFactoryBuilder().build(inputStream);
}


@Test
public void test() {
  // 1、根据配置文件获取 sqlSessionFactory 对象
  try {
    SqlSessionFactory sqlSessionFactory = getSqlSessionFactory();
    // 2、使用 sqlSessionFactory 获取 sqlSession 对象
    // sqlSession 不是线程安全的，所以每次使用都必须重新获取且使用完毕之后需要正确关闭
    try (SqlSession openSession = sqlSessionFactory.openSession()) {
      // 3、MyBatis 为接口自动的创建一个代理对象，代理对象去执行增删改查方法
      UserMapper mapper = openSession.getMapper(UserMapper.class);
      User user = mapper.getUserById(1);
      System.out.println(mapper.getClass());
      System.out.println(user);
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```



