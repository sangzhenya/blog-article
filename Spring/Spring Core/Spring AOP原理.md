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

**首先看一下获取 `BeanFactory` 的过程如下：**

启动容器的时候会调用 `refresh` 方法刷新容器，在刷新的过程中调用 `registerBeanPostProcessors` 方法，注册 Bean 的后置处理器用来拦截 Bean 的创建，其直接调用 `PostProcessorRegistrationDelegate.registerBeanPostProcessors` 去注册。找到容器中所有 `BeanPostProcessor` 类型的后置处理器名称， 依次分到 `priorityOrderedPostProcessors`， `orderedPostProcessorNames`， `nonOrderedPostProcessorNames` 去注册，最终都会放到 `internalPostProcessors` 中。这里会去注册一个 `org.springframework.aop.config.internalAutoProxyCreator` 的 Bean 对象 `AnnotationAwareAspectJAutoProxyCreator` 属于 `orderedPostProcessorNames`，这个对象的定义信息在之前已经通过 `AspectJAutoProxyRegistrar` 放入到容器中了，注册的过程会获取到这个 Bean。

```java
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
for (String ppName : orderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    orderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
        internalPostProcessors.add(pp);
    }
}
```

应为目前容器中还没有这个 Bean 对象，所以会去走 Create Bean 的流即 `getBean` 方法调用 `doGetBean` 方法，继续调用 `getSingleton` 方法，还是没有找到，所以去调用 `createBean` 方法去创建 Bean，调用 `doCreateBean` 方法 ，在该方法中创建 Bean 的包装实例并且为属性赋值，最后调用 `initializeBean` 去初始化 Bean 对象，在此过程中会去调用 `invokeAwareMethods` 方法中根据不同的 `Aware` 为 Bean 设置不同的属性。

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

调用 `AbstractAdvisorAutoProxyCreator` 的 `setBeanFactory` 首先执行其父类`AbstractAutoProxyCreator` 中的 `setBeanFactory` 设置 `beanFactory`。然后调用 `AbstractAdvisorAutoProxyCreator` 自己的 `initBeanFactory`将 `BeanFactory` 包装成一个 `BeanFactoryAdvisorRetrievalHelperAdapter`。

在 `invokeAwareMethods` 方法执行之后会调用 `applyBeanPostProcessorsBeforeInitialization` 在 Init Bean 之前，然后 Init Bean，之后再调用 `applyBeanPostProcessorsAfterInitialization` 方法，两个 `applyBeanPostProcessors` 方法都是找到所有的 `BeanPostProcessor` 分别调用其 `postProcessBeforeInitialization` 和 `postProcessAfterInitialization` 方法。

最后会把所有的的 `internalPostProcessors` 都放到 `BeanFactory` 中。

```java
private static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {
    for (BeanPostProcessor postProcessor : postProcessors) {
        beanFactory.addBeanPostProcessor(postProcessor);
    }
}
```

**然后看一下 `AspectJAwareAdvisorAutoProxyCreator` 作为 `BeanPostProcessor` 执行的过程：**

