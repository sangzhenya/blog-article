---
title: "Spring 注解 --- 初始化、销毁及赋值"
tags: ["Spring", "Spring 注解"]
categories: ["Spring"]
date: "2019-01-20T09:00:00+08:00"
---

bean 初始化和销毁方法可以通过以下四种方式指定：

1. 通过 Spring 提供的 `@Bean` 属性的  `initMethod` 和 `destoryMethod` 指定
2. 通过实现 `InitializingBean`, `DisposableBean` 接口
3. 使用 JSR250 提供的 `@PostConstruct` 和 `@PreDestroy` 注解
4. 使用自定义 `BeanPostProcessor` 

对于单实例对象，容器启动的时候创建对象，创建完成调用初始化方法，容器关闭的时候（`applicationContext.close();`）调用销毁方法

对于多实例对象，每次使用的时候创建对象，创建完成调用初始化方法，不会调用销毁方法。

第一种：使用 `@Bean` 属性

```java
public class Car {
    public Car() {
        System.out.println("Conduction....");
    }

    public void init() {
        System.out.println("Init...");
    }

    public void destroy() {
        System.out.println("Destroy...");
    }
}

@Configuration
public class MainConfigOfLifeCycle {
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Car car() {
        return new Car();
    }
}
```

第二种：实现接口

```java
public class Cat implements InitializingBean, DisposableBean {
    public Cat() {
        System.out.println("Cat conduction...");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("Cat destroy...");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("Cat afterPropertiesSet...");
    }
}
```

第三种：使用 JSR250 提供的注解

```java
public class Dog {
    public Dog() {
        System.out.println("Dog Conduction....");
    }

    @PostConstruct
    public void init() {
        System.out.println("Dog Init...");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("Dog Destroy...");
    }
}
```

第四种：自定义 `BeanPostProcessor`

```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    /**
     * 初始化的方法之前调用
     * @param bean  Bean 对象
     * @param beanName  Bean 名称
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization" + bean + "::" + beanName);
        return bean;
    }

    /**
     * 初始化方法之后调用
     * @param bean  Bean 对象
     * @param beanName  Bean 名称
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization" + bean + "::" + beanName);
        return bean;
    }
}
```



`@PropertySource` 引入 properties 文件。

`@Value("DemoName")`  为属性赋值。可以使用以下几种：

1. 基础数值
2. SpEL 表达式 `#{}`
3. `${}` 取出文件的值，在运行环境中的值。

```java
// demo.properties
dog.name=snoopy

@PropertySource({"classpath:/demo.properties"})
public class Dog {
    @Value("${dog.name}")
    private String name;
}
    

// 同样也可以通过 ApplicationContext 直接获取
ConfigurableEnvironment environment = applicationContext.getEnvironment();
System.out.println(environment.getProperty("dog.name"));
```

