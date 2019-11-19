## Spring AOP 原理

AOP 动态代理，指运行期间动态的将某段代码切入到指定方法指定位置进行编程的方式。主要有前置通知，后置通知，返回通知，异常通知，环绕通知几种。

如果要开启 AOP，需要添加 `@EnableAspectJAutoProxy` 注解。切面类需要添加 `@Aspect` 注解如下：

```java
@Aspect
@Component
public class LogAspects {
    @Pointcut("execution(public int com.xinyue.myspring.aop.MyMessage.*(..))")
    public void pointCut() {}
```

具体 Spring 实现 AOP 的原理如下：

`@EnableAspectJAutoProxy` 注解中引入了 `AspectJAutoProxyRegistrar` 类，其实现了 `ImportBeanDefinitionRegistrar` 接口，在 `registerBeanDefinitions()` 方法中为容器引入 `AnnotationAwareAspectJAutoProxyCreator` 类。

```java
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
    BeanDefinitionRegistry registry, @Nullable Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

@Nullable
private static BeanDefinition registerOrEscalateApcAsRequired(
    Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        return null;
    }

    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}
```

`AnnotationAwareAspectJAutoProxyCreator` 实现了 `BeanPostProcessor` 是一个后置处理器，且实现了`BeanFactoryAware` 可以拿到 Bean 工厂。

![类结构]( http://img.sangzhenya.com/Snipaste_2019-11-19_23-08-05.png )