# Spring Boot + MyBatis 多数据源配置

主要有两种配置方式：第一种是自定义配置 `SqlSessionFactory` ，第二种是使用 `AbstractRoutingDataSource  + AOP` 的方式实现

## 自定义 SqlSessionFactory

首先关闭 Spring Boot 对于 `DataSourceAutoConfiguration`:

```java
@SpringBootApplication(exclude = {
        DataSourceAutoConfiguration.class
})
```

然后配置 DataSourceConfig，例如这边有两个数据库连接：

```java
@Configuration
public class DataSourceConfig {
    @Bean(name = "db1")
    @ConfigurationProperties(prefix = "spring.datasource.db1")
    public DataSource db1Source() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "db2")
    @ConfigurationProperties(prefix = "spring.datasource.db2")
    public DataSource db2Source() {
        return DataSourceBuilder.create().build();
    }
}
```

然后在配置文件中分别添加两个数据库的连接地址：

```yaml
spring:
  datasource:
    db1:
      jdbc-url: ${db1url}
      username: ${db1name}
      password: ${db1password}
    db2:
      jdbc-url: ${db2url}
      username: ${db2name}
      password: ${db2password}
```

然后分别配置不同 DB 的 `SqlSessionFactory`, 并指定扫描的包 :

```java
@Configuration
@MapperScan(basePackages = {"com.xinyue.multi.source.mapper.db1"}, sqlSessionFactoryRef = "sqlSessionFactoryDb1")
public class Db1Config {

    @Autowired
    private DataSource db1;

    @Bean
    public SqlSessionFactory sqlSessionFactoryDb1() throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(db1);
        return factoryBean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplateDb1() throws Exception {
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryDb1();
        sqlSessionFactory.getConfiguration().setMapUnderscoreToCamelCase(true);
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

```java
@Configuration
@MapperScan(basePackages = {"com.xinyue.multi.source.mapper.db2"}, sqlSessionFactoryRef = "sqlSessionFactoryDb2")
public class Db2Config {

    @Autowired
    private DataSource db2;

    @Bean
    public SqlSessionFactory sqlSessionFactoryDb2() throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(db2);
        return factoryBean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplateDb2() throws Exception {
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryDb2();
        sqlSessionFactory.getConfiguration().setMapUnderscoreToCamelCase(true);
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

然后编写 Mapper 

```java
package com.xinyue.multi.source.mapper.db1;

@Mapper
public interface ArticleMapper {
    @Select("SELECT * FROM article WHERE id = #{id}")
    Article findById(int id);
}
```

```java
package com.xinyue.multi.source.mapper.db2;

@Mapper
public interface UserMapper {
    @Select("SELECT * FROM user WHERE id = #{id}")
    User findById(int id);
}
```

然后编写测试的 Service 和 Controller

```java
public class DemoServiceImpl implements DemoService {
    private final ArticleMapper articleMapper;
    private final UserMapper userMapper;

    @Autowired
    public DemoServiceImpl(ArticleMapper articleMapper, UserMapper userMapper) {
        this.articleMapper = articleMapper;
        this.userMapper = userMapper;
    }

    @Override
    public Article findArticleById(int id) {
        return articleMapper.findById(id);
    }

    @Override
    public User findUserById(int id) {
        return userMapper.findById(id);
    }
}
```

```java
@RestController
public class DemoController {
    private final DemoService demoService;

    public DemoController(DemoService demoService) {
        this.demoService = demoService;
    }

    @GetMapping(value = "/article/{id}")
    public Article getArticleInfo(@PathVariable("id") int id) {
        return demoService.findArticleById(id);
    }

    @GetMapping(value = "/user/{id}")
    public User getUserInfo(@PathVariable("id") int id) {
        return demoService.findUserById(id);
    }
}
```

然后通过浏览器访问连接测试查询 article 的时候则使用 db1 的配置，而查 user 的使用 db2 的配置。

使用 DB1 查询 Article

![https://i.loli.net/2020/05/31/LaA2iWNywnfUcVJ.png](https://i.loli.net/2020/05/31/LaA2iWNywnfUcVJ.png)

使用 DB2 查询 User

![https://i.loli.net/2020/05/31/9JYvQWjZgIPU186.png](https://i.loli.net/2020/05/31/9JYvQWjZgIPU186.png)

## AbstractRoutingDataSource  + AOP

同样也要先关闭 Spring Boot 对于 `DataSourceAutoConfiguration`:

```java
@SpringBootApplication(exclude = {
        DataSourceAutoConfiguration.class
})
```

同样添加 `DataSourceConfig` 有两个 DB 的 Config：

```java
@Configuration
public class DataSourceConfig {
    @Bean(name = "db1")
    @ConfigurationProperties(prefix = "spring.datasource.db1")
    public DataSource db1Source() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "db2")
    @ConfigurationProperties(prefix = "spring.datasource.db2")
    public DataSource db2Source() {
        return DataSourceBuilder.create().build();
    }
}
```

同样在配置文件中配置数据库的链接信息：

```yaml
spring:
  datasource:
    db1:
      jdbc-url: ${db1url}
      username: ${db1name}
      password: ${db1password}
    db2:
      jdbc-url: ${db2url}
      username: ${db2name}
      password: ${db2password}
```

然后添加 `DbConfig` Config，配置 `dataSource`， 设置默认使用 db1 的配置:

```java
@Configuration
public class DbConfig {
    public final static String DB_1_NAME = "db1";
    public final static String DB_2_NAME = "db2";

    @Autowired
    private DataSource db1;
    @Autowired
    private DataSource db2;

    @Bean
    @Primary
    public DataSource dataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>(2);
        targetDataSources.put(DB_1_NAME, db1);
        targetDataSources.put(DB_2_NAME, db2);
        System.out.println("DataSources:" + targetDataSources);
        return new DynamicDataSource(db1, targetDataSources);
    }
}
```

添加 DynamicDataSource 继承 AbstractRoutingDataSource 用于决定使用哪个 dataSource:

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources) {
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return getDataSource();
    }

    public static void setDataSource(String dataSource) {
        CONTEXT_HOLDER.set(dataSource);
    }

    public static String getDataSource() {
        return CONTEXT_HOLDER.get();
    }

    public static void clearDataSource() {
        CONTEXT_HOLDER.remove();
    }
}
```

然后添加一个自定义的注解用于设置不同方法或者类使用的 `dataSource`:

```java
@Documented
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DataSourceManage {
    String value() default DbConfig.DB_1_NAME;
}
```

然后编写 AOP 拦截使用了上述注解的方法，并为方法注入 dataSource:

```java
@Aspect
@Component
@Order(value = 1)
public class DataSourceAspect {
    private final Log log = LogFactory.getLog(getClass());

    @Pointcut("@annotation(com.xinyue.multi.source.anno.DataSourceManage)")
    public void dataSourcePointCut1() {}
    @Pointcut("@within(com.xinyue.multi.source.anno.DataSourceManage)")
    public void dataSourcePointCut2() {}

    @Around("dataSourcePointCut1() || dataSourcePointCut2()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();
        DataSourceManage ds = method.getAnnotation(DataSourceManage.class);
        if (ds == null) {
            ds = point.getTarget().getClass().getAnnotation(DataSourceManage.class);
        }
        if (ds != null) {
            DynamicDataSource.setDataSource(ds.value());
            log.info("set datasource is " + ds.value());
        }
        try {
            return point.proceed();
        } finally {
            DynamicDataSource.clearDataSource();
            log.info("clean datasource");
        }
    }
}
```

Mapper 文件上第一种配置相同：

```java
@Mapper
public interface ArticleMapper {
    @Select("SELECT * FROM article WHERE id = #{id}")
    Article findById(int id);
}
```

```java
@Mapper
public interface UserMapper {
    @Select("SELECT * FROM user WHERE id = #{id}")
    User findById(int id);
}
```

Service 方法稍作修改，为需要自定义 dataSource 的方法上添加上自定义的注解， Controller 不变：

```java
@Service
public class DemoServiceImpl implements DemoService {
    private final ArticleMapper articleMapper;
    private final UserMapper userMapper;

    @Autowired
    public DemoServiceImpl(ArticleMapper articleMapper, UserMapper userMapper) {
        this.articleMapper = articleMapper;
        this.userMapper = userMapper;
    }

    @Override
    public Article findArticleById(int id) {
        return articleMapper.findById(id);
    }

    @DataSourceManage(value = DbConfig.DB_2_NAME)
    @Override
    public User findUserById(int id) {
        return userMapper.findById(id);
    }
}
```

```java
@RestController
public class DemoController {
    private final DemoService demoService;
    private final UserService userService;

    public DemoController(DemoService demoService) {
        this.demoService = demoService;
    }

    @GetMapping(value = "/user/{id}")
    public User getUserInfo(@PathVariable("id") int id) {
        return demoService.findUserById(id);
    }

    @GetMapping(value = "/article/{id}")
    public Article getArticleInfo(@PathVariable("id") int id) {
        return demoService.findArticleById(id);
    }
}
```

此外如果需要自定义 MyBatis 可以添加一个 `mybatisConfigurationCustomizer`：

```java
@Configuration
public class MyIBatisCosomizer {
    private final Log log = LogFactory.getLog(getClass());
    @Bean
    ConfigurationCustomizer mybatisConfigurationCustomizer() {
        log.info("Use customizer to customized mybaits");
        return configuration -> {
            // Camel case
            configuration.setMapUnderscoreToCamelCase(true);
        };
    }
}
```

然后便可以测试分别访问 `http://localhost:8080/article/1` 和 `http://localhost:8080/user/1` 的结果，同样是查询 Article 使用 db1 的配置，查询 User 使用 DB2 的配置。

最后附录一下 pom.xml 文件中依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.xinyue</groupId>
    <artifactId>multisource</artifactId>
    <version>1.0-SNAPSHOT</version>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.0.RELEASE</version>
        <relativePath/>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.11</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.0.1</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```