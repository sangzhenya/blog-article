---
title: "Spring ApplicationListener 和 ApplicationEvent"
tags: ["Spring", "源码"]
categories: ["Spring"]
date: "2019-01-14T09:00:00+08:00"
---

用于监听容器中发布的事件进而完成事件驱动模型的开发，可以自定义一个 `ApplicationListener` 用于监听容器中发布的事件。一个简单的自定义的 ApplicationListener 如下：
```java
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {
    private Log log = LogFactory.getLog(MyApplicationListener.class);
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        log.info("event::" + event);
    }
}
```

会打出下面两个事件：

```log
信息: Initializing c3p0-0.9.5.4 [built 23-March-2019 23:00:48 -0700; debug? true; trace: 10]
11月 25, 2019 8:39:13 下午 com.xinyue.myspring.oth.MyApplicationListener onApplicationEvent
信息: class::class org.springframework.context.event.ContextRefreshedEvent
11月 25, 2019 8:39:13 下午 com.xinyue.myspring.oth.MyApplicationListener onApplicationEvent
信息: class::class org.springframework.context.event.ContextClosedEvent
```

包含容器刷新完成的时间和容器关闭的事件，预设的几种 ApplicationEvent 如下图所示：

![几种 ApplicationEvent]( http://img.programya.com/Snipaste_2019-11-25_21-15-17.png )

也可以自己添加自己定义的事件，如下所示：
```java
applicationContext.publishEvent(new ApplicationEvent("") {});
```

再运行会打印出 `信息: event::com.xinyue.myspring.tx.OthTest$1[source=OthTest]`

接下看一下容器中默认代刷新完成事件和关闭事件分别是什么时候产生调用的。首先看一下容器刷新完成事件：

容器启动的时候会去调用 `refresh` 方法刷新容器，在刷新的最后一步调用 `finishRefresh` 方法完成容器的刷新，如下所示：

```java
protected void finishRefresh() {
    // 清理缓存
    // Clear context-level resource caches (such as ASM metadata from scanning).
    clearResourceCaches();

    // Initialize lifecycle processor for this context.
    initLifecycleProcessor();

    // Propagate refresh to lifecycle processor first.
    getLifecycleProcessor().onRefresh();

    // 发布容器刷新完成事件
    // Publish the final event.
    publishEvent(new ContextRefreshedEvent(this));

    // Participate in LiveBeansView MBean, if active.
    LiveBeansView.registerApplicationContext(this);
}
```

其中发布事件具体如下：

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");

    // 包装 ApplicationEvent
    // Decorate event as an ApplicationEvent if necessary
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent) event;
    }
    else {
        applicationEvent = new PayloadApplicationEvent<>(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
        }
    }

    // Multicast right now if possible - or lazily once the multicaster is initialized
    if (this.earlyApplicationEvents != null) {
        // 获取早期事件
        this.earlyApplicationEvents.add(applicationEvent);
    }
    else {
        // 获取 applicationEvent 的派发器
        getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
    }

    // 如果当前 context 有 parent 则通过 parent 发布事件
    // Publish event via parent context as well...
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
        }
        else {
            this.parent.publishEvent(event);
        }
    }
}
```

```java
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    Executor executor = getTaskExecutor();
    // 得到所有的 ApplicationListener
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            // 逐个 invoke listener 方法
            // 本质上也就是调用每个 Listener 的 onApplicationEvent(event); 方法
            invokeListener(listener, event);
        }
    }
}
```

获取 `ApplicationListeners` 的方法如下：

```java
protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

    Object source = event.getSource();
    Class<?> sourceType = (source != null ? source.getClass() : null);
    // 根据 sourceType 和 eventType 生成缓存 Key
    ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

    // 首先从 cache 中拿
    // Quick check for existing entry on ConcurrentHashMap...
    ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
    if (retriever != null) {
        return retriever.getApplicationListeners();
    }

    if (this.beanClassLoader == null ||
        (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
         (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
        // Fully synchronized building and caching of a ListenerRetriever
        synchronized (this.retrievalMutex) {
            // 从获取 ListenerRetriever
            retriever = this.retrieverCache.get(cacheKey);
            // 如果不为空则直接返回缓存的
            if (retriever != null) {
                return retriever.getApplicationListeners();
            }
            // 新建一个用于缓存的 ListenerRetriever
            retriever = new ListenerRetriever(true);
            // 获取所有Listener 并放到 缓存中
            Collection<ApplicationListener<?>> listeners =
                retrieveApplicationListeners(eventType, sourceType, retriever);
            this.retrieverCache.put(cacheKey, retriever);
            return listeners;
        }
    }
    else {
        // 如果没有设置 ListenerRetriever cache 则直接尝试去拿
        // No ListenerRetriever caching -> no synchronization necessary
        return retrieveApplicationListeners(eventType, sourceType, null);
    }
}
```

```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
			ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {

    List<ApplicationListener<?>> allListeners = new ArrayList<>();
    Set<ApplicationListener<?>> listeners;
    Set<String> listenerBeans;
    synchronized (this.retrievalMutex) {
        // 获取在容器中注册的 ApplicationListeners
        listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
        listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
    }

    // 遍历并存放到用于缓存的 retriever 中
    // Add programmatically registered listeners, including ones coming
    // from ApplicationListenerDetector (singleton beans and inner beans).
    for (ApplicationListener<?> listener : listeners) {
        if (supportsEvent(listener, eventType, sourceType)) {
            if (retriever != null) {
                retriever.applicationListeners.add(listener);
            }
            allListeners.add(listener);
        }
    }

    // Add listeners by bean name, potentially overlapping with programmatically
    // registered listeners above - but here potentially with additional metadata.
    if (!listenerBeans.isEmpty()) {
        ConfigurableBeanFactory beanFactory = getBeanFactory();
        // 遍历 Listener Bean
        for (String listenerBeanName : listenerBeans) {
            try {
                // 判断是否支持当前的 event
                if (supportsEvent(beanFactory, listenerBeanName, eventType)) {
                    // 如果支持则拿到当前的 Event
                    ApplicationListener<?> listener =
                        beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                    if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
                        if (retriever != null) {
                            // 如果当前的 Listener 还没有添加到 retriever 则加入到 retriever 中
                            if (beanFactory.isSingleton(listenerBeanName)) {
                                // 如果是单实例的则把 listener 放到 retriever.applicationListeners 中
                                retriever.applicationListeners.add(listener);
                            }
                            else {
                                // 否则放到 retriever.applicationListenerBeans 中
                                retriever.applicationListenerBeans.add(listenerBeanName);
                            }
                        }
                        allListeners.add(listener);
                    }
                }
                else {
                    // Remove non-matching listeners that originally came from
                    // ApplicationListenerDetector, possibly ruled out by additional
                    // BeanDefinition metadata (e.g. factory method generics) above.
                    // 如果不支持当前的 event 则从 retriever
                    Object listener = beanFactory.getSingleton(listenerBeanName);
                    if (retriever != null) {
                        retriever.applicationListeners.remove(listener);
                    }
                    allListeners.remove(listener);
                }
            }
            catch (NoSuchBeanDefinitionException ex) {
                // Singleton listener instance (without backing bean definition) disappeared -
                // probably in the middle of the destruction phase
            }
        }
    }

    AnnotationAwareOrderComparator.sort(allListeners);
    // 如果 retriever 的 applicationListenerBeans 为空则说明所有支持的都
    // 已经放到 allListeners 中且是单实例的所以直接将 applicationListeners 清空然后放入 allListeners
    if (retriever != null && retriever.applicationListenerBeans.isEmpty()) {
        retriever.applicationListeners.clear();
        retriever.applicationListeners.addAll(allListeners);
    }
    return allListeners;
}
```

从缓存中拿 Listener 流程如下：

```java
public Collection<ApplicationListener<?>> getApplicationListeners() {
    List<ApplicationListener<?>> allListeners = new ArrayList<>(
        this.applicationListeners.size() + this.applicationListenerBeans.size());
    allListeners.addAll(this.applicationListeners);
    // 如果 applicationListenerBeans 不为空则表明有多实例的
    if (!this.applicationListenerBeans.isEmpty()) {
        BeanFactory beanFactory = getBeanFactory();
        for (String listenerBeanName : this.applicationListenerBeans) {
            try {
                // 从容器中获取一个 bean 放到 allListeners 中
                ApplicationListener<?> listener = beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                if (this.preFiltered || !allListeners.contains(listener)) {
                    allListeners.add(listener);
                }
            }
            catch (NoSuchBeanDefinitionException ex) {
                // Singleton listener instance (without backing bean definition) disappeared -
                // probably in the middle of the destruction phase
            }
        }
    }
    if (!this.preFiltered || !this.applicationListenerBeans.isEmpty()) {
        AnnotationAwareOrderComparator.sort(allListeners);
    }
    // 返回所有的 Listener
    return allListeners;
}
}
```



容器关闭的事件流程如下：

调用 `close` 方法关闭容器，同步的调用 `doClose` 执行关闭过程，在 其中会调用 `publishEvent(new ContextClosedEvent(this));` 发布关闭容器的事件。发布事件的具体流程和刷新容器的一致，即获取到所有的 Listener 逐个调用其 `onApplicationEvent(event);` 方法。

此外 `publishEvent` 方法中是通过 `getApplicationEventMulticaster` 拿到派发器之后发送的，其初始化流程如下：

在容器启动的时候会刷新容器 `refresh` 方法中 调用 `initApplicationEventMulticaster` 初始化派发器。如下所示：

```java
protected void initApplicationEventMulticaster() {
    // 获取到 beanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 如果 BeanFactory 中已经有了派发器，则直接获取出来即可
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        // 否则就新建一个 SimpleApplicationEventMulticaster 放到容器中
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                         "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
    }
}
```

容器中监听器的注册时 `refresh ` 中调用的 `registerListeners` 方法如下：

```java
protected void registerListeners() {
    // 先注册特殊的 ApplicationListener
    // Register statically specified listeners first.
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // 然后从容器中获取所有的 ApplicationListener
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        // 逐个添加到派发器中
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }

    // 在获取到派发器之后发布早起的事件
    // Publish early application events now that we finally have a multicaster...
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

```java
// 将 ApplicationListener 添加到派发器中
public void addApplicationListener(ApplicationListener<?> listener) {
    synchronized (this.retrievalMutex) {
        // 对于代理要去除，避免多次 invoke 同一个 listener
        // Explicitly remove target for a proxy, if registered already,
        // in order to avoid double invocations of the same listener.
        Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
        if (singletonTarget instanceof ApplicationListener) {
            this.defaultRetriever.applicationListeners.remove(singletonTarget);
        }
        // 将 plicationListener 添加到容器中
        this.defaultRetriever.applicationListeners.add(listener);
        this.retrieverCache.clear();
    }
}
```

此外除了通过实现接口的方式监听事件之外还可以通过注解的方式实现如下所示：

```java
@Service
public class MyListenerService {
    private Log log = LogFactory.getLog(MyListenerService.class);
    @EventListener(classes = ApplicationContextEvent.class)
    public void listener(ApplicationContextEvent event) {
        log.info(event);
    }
}
```

其原理如下：

```java
// @EventListener 
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EventListener {
```

使用 `EventListenerMethodProcessor` 处理该注解，该类实现了 `SmartInitializingSingleton` 接口。该接口是在所有的 单示例 Bean 已经创建完成执行。看一下调用栈发现。同样启动容器先是 `refresh` 刷新容器，在其倒数第二步调用 `finishBeanFactoryInitialization` 方法初始化所有剩余的单示例 bean，调用 `beanFactory.preInstantiateSingletons();` 去初始化所有的单示例 bean。其中创建完所有 Bean 之后接下来就是调用 `SmartInitializingSingletonafterSingletonsInstantiated` 接口

```java
// Trigger post-initialization callback for all applicable beans...
for (String beanName : beanNames) {
    // 获取 bean 实例
    Object singletonInstance = getSingleton(beanName);
    // 判断是否是 SmartInitializingSingleton
    if (singletonInstance instanceof SmartInitializingSingleton) {
        final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                smartSingleton.afterSingletonsInstantiated();
                return null;
            }, getAccessControlContext());
        }
        else {
            // 触发 afterSingletonsInstantiated 方法
            smartSingleton.afterSingletonsInstantiated();
        }
    }
}
```

对于 ``EventListenerMethodProcessor` ` 而言如下流程：

```java
@Override
public void afterSingletonsInstantiated() {
    // 获取 BeanFactory
    ConfigurableListableBeanFactory beanFactory = this.beanFactory;
    Assert.state(this.beanFactory != null, "No ConfigurableListableBeanFactory set");
    String[] beanNames = beanFactory.getBeanNamesForType(Object.class);
    // 获取所有的 Bean
    for (String beanName : beanNames) {
        if (!ScopedProxyUtils.isScopedTarget(beanName)) {
            Class<?> type = null;
            try {
                type = AutoProxyUtils.determineTargetClass(beanFactory, beanName);
            }
            catch (Throwable ex) {
                // An unresolvable bean type, probably from a lazy bean - let's ignore it.
                if (logger.isDebugEnabled()) {
                    logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
                }
            }
            if (type != null) {
                if (ScopedObject.class.isAssignableFrom(type)) {
                    try {
                        Class<?> targetClass = AutoProxyUtils.determineTargetClass(
                            beanFactory, ScopedProxyUtils.getTargetBeanName(beanName));
                        if (targetClass != null) {
                            type = targetClass;
                        }
                    }
                    catch (Throwable ex) {
                        // An invalid scoped proxy arrangement - let's ignore it.
                        if (logger.isDebugEnabled()) {
                            logger.debug("Could not resolve target bean for scoped proxy '" + beanName + "'", ex);
                        }
                    }
                }
                try {
                    // 处理 bean
                    processBean(beanName, type);
                }
                catch (Throwable ex) {
                    throw new BeanInitializationException("Failed to process @EventListener " +
                                                          "annotation on bean with name '" + beanName + "'", ex);
                }
            }
        }
    }
}
```

```java
private void processBean(final String beanName, final Class<?> targetType) {
    // 判断是否应该被处理
    if (!this.nonAnnotatedClasses.contains(targetType) &&
        AnnotationUtils.isCandidateClass(targetType, EventListener.class) &&
        !isSpringContainerClass(targetType)) {

        // 获取所有有注解的方法
        Map<Method, EventListener> annotatedMethods = null;
        try {
            annotatedMethods = MethodIntrospector.selectMethods(targetType,
                                                                (MethodIntrospector.MetadataLookup<EventListener>) method ->
                                                                AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
        }
        catch (Throwable ex) {
            // An unresolvable type in a method signature, probably from a lazy bean - let's ignore it.
            if (logger.isDebugEnabled()) {
                logger.debug("Could not resolve methods for bean with name '" + beanName + "'", ex);
            }
        }

        if (CollectionUtils.isEmpty(annotatedMethods)) {
            this.nonAnnotatedClasses.add(targetType);
            if (logger.isTraceEnabled()) {
                logger.trace("No @EventListener annotations found on bean class: " + targetType.getName());
            }
        }
        else {
            // Non-empty set of methods
            ConfigurableApplicationContext context = this.applicationContext;
            Assert.state(context != null, "No ApplicationContext set");
            List<EventListenerFactory> factories = this.eventListenerFactories;
            Assert.state(factories != null, "EventListenerFactory List not initialized");
            // 遍历所有有注解的方法
            for (Method method : annotatedMethods.keySet()) {
                // 遍历 factories
                for (EventListenerFactory factory : factories) {
                    // 判断 factory 是否值当前的 method
                    if (factory.supportsMethod(method)) {
                        // 如果支持将 Method 包装成一个 applicationListener
                        Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
                        ApplicationListener<?> applicationListener =
                            factory.createApplicationListener(beanName, targetType, methodToUse);
                        if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                            ((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
                        }
                        // 将包装后的 Method 放到容器
                        context.addApplicationListener(applicationListener);
                        break;
                    }
                }
            }
            if (logger.isDebugEnabled()) {
                logger.debug(annotatedMethods.size() + " @EventListener methods processed on bean '" +
                             beanName + "': " + annotatedMethods);
            }
        }
    }
}
```





