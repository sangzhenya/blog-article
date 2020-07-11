---
title: "Spring 注解 --- 组件注册"
tags: ["Spring", "Spring 注解"]
categories: ["Spring"]
date: "2019-01-19T09:00:00+08:00"
---

`@Configuration` 把当前文件用作配置类。

```java
@Configuration
public class MainConfig {
```

`@ComponentScan` 指定扫描的包。也可以指定包含或者排除规则。

`includeFilters` 指定包含规则，这时需要使用 `useDefaultFilters = false`。

`excludeFilters` 指定过滤规则。

`type = FilterType.CUSTOM, classes = {MyTypeFilter.class}` 设置自定义规则

```java
// 1. 包扫描，指定扫描的包并设置过滤规则
@Configuration
@ComponentScan(value = "com.xinyue.config",
        excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,
                classes = {Controller.class, Service.class})})
public class MainConfig {}

// 2. 包扫描，指定扫描的包并设置包含规则，并自定义过滤规则。
@Configuration
@ComponentScan(value = "com.xinyue.config",
        includeFilters = {@ComponentScan.Filter(type = FilterType.CUSTOM, classes = {MyTypeFilter.class})},
        useDefaultFilters = false)
public class MainConfig {}

// 自定义过滤规则
public class MyTypeFilter implements TypeFilter {
    /**
     * 判断是否匹配
     * @param metadataReader    读取到当前正在扫描类的信息
     * @param metadataReaderFactory 可以获取其他任何类的信息
     */
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        // 当前类的注解信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
        // 当前正在扫描类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        // 获取当前类资源
        Resource resource = metadataReader.getResource();
        // 获取当前类名
        String className = classMetadata.getClassName();
        return true;
    }
}

```

`@Bean`  给容器注册一个 Bean。类型为返回值类型，id 默认为**方法名**。也可以使用 `value` 属性指定 beanId 例如 `@Bean("demoUser")`。

`@Scope`设置Bean实例化的模式， 常用的是`ConfigurableBeanFactory#SCOPE_PROTOTYPE` 和 `ConfigurableBeanFactory#SCOPE_SINGLETON` 两种类型，默认使用的是：`SCOPE_SINGLETON` 。 如果是单示例的情况会在**项目启动的时候创建对象，否则是每次使用的时候创建对象**。

`@Lazy` 让单实例的 Bean 在第一次使用的时候创建对象，而不是在应用启动的时候去创建。

`@Conditional` 判断是否加载某个 Bean 到容器之中。可以自定义判断规则。

```java
@Configuration
public class MainConfig {
    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public Cat cat() {
        return new Cat();
    }
    @Bean
    @Lazy
    public Dog dog() {
        return new Dog();
    }
    @Bean
    @Conditional({WindowsCondition.class})
    public User windowsUser() {
        return new User();
    }
}

// 自定义过滤规则
public class WindowsCondition implements Condition {
    /**
     * @param context   判断条件能使用的上下文信息
     * @param metadata  注释信息
     */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 可以获取 IOC 使用的 BeanFactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        // 获取类加载器
        ClassLoader classLoader = context.getClassLoader();
        // 获取当前环境信息
        Environment env = context.getEnvironment();
        // 获取到 Bean的注册信息
        BeanDefinitionRegistry registry =  context.getRegistry();
        return true;
    }
}   
    
```

`@Import` 像容器中添加一个 Bean。也可以使用 `ImportSelector` 和 `ImportBeanDefinitionRegistrar` 自定义导入规则。

```java
@Configuration
@Import({MyImportSelector.class, 
         MyImportBeanDefinitionRegistrar.class, User.class})
public class MainConfig {}

// 通过实现 ImportSelector 实现自定义导入规则
public class MyImportSelector implements ImportSelector {
    /**
     * @param importingClassMetadata  当前 @Import 注解的类的所以注解信息
     * @return 返回需要导入的全类名数组
     */
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"model.Blue", "model.Red"};
    }
}

// 通过实现 ImportBeanDefinitionRegistrar 自定义导入规则
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    /**
     * @param importingClassMetadata    当前类的注解信息
     * @param registry  BeanDefinition 注册类；使用 BeanDefinitionRegistry 将 Bean 注册到容器中。
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        if (!registry.containsBeanDefinition("color")) {
            // （注册名称，注册类）
            registry.registerBeanDefinition("color", new RootBeanDefinition(Color.class));
        }
    }
}

```

`FactoryBean` 注册 Bean 到 容器中。可以直接调用 `applicationContext.getBean("black")` 获取 Bean，如果获取 FactoryBean 则需要可以使用 `applicationContext.getBean("&black")`

```java
// FactoryBean
public class BlackFactoryBean implements FactoryBean<Black> {
    /**
     * @return  返回对象添加到容器中
     */
    @Override
    public Black getObject() throws Exception {
        return new Black();
    }
    @Override
    public Class<?> getObjectType() {
        return null;
    }
    /**
     * @return true 表示单实例
     */
    @Override
    public boolean isSingleton() {
        return false;
    }
}
// Config 类
@Configuration
public class MainConfig {
    @Bean
    public BlackFactoryBean black() {
        return new BlackFactoryBean();
    }
}
```