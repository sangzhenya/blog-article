## Spring 注解 --- 自动注入

`@Autowired` 自动注入对象。

1. 默认按照类型查找类似: `applicationContext.getBean(UserDao.class)`
2. 如果找到多个，则将属性名称作为 Bean Id 去容器中查找：`applicationContext.getBean('userDao')`

另外也可以在定义 bean 的地方添加 `@Primary` 注解，作为默认注入的 bean。也可以使用 `@Qualifier` 注解，和`@Autowired` 放在一起， 指定注入的组件，比 `@Primary` 优先级更高。

使用自动注入时如果找不到会抛出异常，所以对于可能找不到的 Bean 可以设置 `@Autowired(required = false)` ，则不会再抛异常。

```java
public interface UserDao {
    String getId();
}

@Primary
@Repository
public class WindowsUserDaoImpl implements UserDao{
    @Override
    public String getId() {
        return this.getClass().getSimpleName();
    }
}
@Repository
public class LinuxUserDaoImpl implements UserDao{
    @Override
    public String getId() {
        return this.getClass().getSimpleName();
    }
}

// 默认使用添加了 @Primary 注解的 Bean
@Service
public class UserService {
    private final UserDao userDao;

    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}

// 优先使用 @Qualifier 指定的 Bean
@Service
public class UserService {
    private final UserDao userDao;

    public UserService(@Qualifier("linuxUserDaoImpl") UserDao userDao) {
        this.userDao = userDao;
    }
}

```



对于自动注入也可以使用 Java 规范中 `JSR250` 中的 `@Resource` 和 `JSR330` 中的 `@Inject` 自动注入。

1. 对于 `@Resource` 不支持 Spring 的注解 `@Primary` 和 `required=false` 属性。

2. 对于 `@Inject` 需要额外导入 `javax.inject` 包，不支持 `required=false` 属性。

   ```xml
   <dependency>
       <groupId>javax.inject</groupId>
       <artifactId>javax.inject</artifactId>
       <version>1</version>
   </dependency>
   ```

   

`@Autowired`  也可以标注以下地方： 

1. 构造器: 如果只有一个有参构造器，可以省略 `@Autowired` 。参数位置的组件会自动从容器中获取。
2. *参数*
3. 方法：如果在 `@Bean` 创建的时候，方法参数值也会从容器中获取，可以忽略 `@Autowired` 注解。
4. 属性



如果自定义组件想要使用 Spring 容器底层的一些组件（`ApplicationContext`, `BeanFactory ` 等等）。自定义组件可以实现 `xxxAware` ，在创建对象的时候，会调用接口规定的方法注入相关的组件。例如：

1. `ApplicaitonContextAware`
2. `ApplicationEventPublisherAware`
3. `BeanFactoryAware`
4. `EnvironmentAware`
5. `ResourceLoadAware`
6. `BeanNameAware`
7. `EmbeddeValueResolverAware`

对于任意一种 `xxxAware` 都有对应的 `xxxAwareProcessor` 去处理。例如 `ApplicationContextAware` 由 `ApplicationContextAwareProcessor` 处理。



`@Profile `为不同的环境注入不同的组件，可以标注在方法上，也可以标注在类上。如果未标注则表示未指定 Profile 的时候加载，等于 `default`

```java
@Profile	// 等同于 @Profile("default")  
@Bean("user")
public User defaultUser() {
    User user = new User();
    user.setName("DEFAULT");
    return user;
}

@Profile("dev")
@Bean("user")
public User devUser() {
    User user = new User();
    user.setName("DEV");
    return user;
}

@Profile("prd")
@Bean("user")
public User prdUser() {
    User user = new User();
    user.setName("PRD");
    return user;
}
```

可以通过以下两种方式指定启用的 Profile

```powershell
#1. VM arguments 指定启用的 Profile 名称
-Dspring.profiles.active=dev

#2. 通过代码指定 
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
applicationContext.getEnvironment().setActiveProfiles("dev");
applicationContext.register(MainConfig.class);
applicationContext.refresh();
```

