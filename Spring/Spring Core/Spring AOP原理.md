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

被切入的 Message 类如下：

```java
@Component
public class MyMessage {
    public int sayHello(int count) {
        System.out.println("Hello World::" + count);
        return count + 1;
    }
}
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

主要有两个方法：

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    // 获取 cache key，正常来说就是 beanName
    Object cacheKey = getCacheKey(beanClass, beanName);

    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        /
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        // 如果是PointCut, Advisor 等切面用的类则不需要代理
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }

    // Create proxy here if we have a custom TargetSource.
    // Suppresses unnecessary default instantiation of the target bean:
    // The TargetSource will handle target instances in a custom fashion.
    // 创建代理对象，首先尝试获取 Custom Traged Source
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    // 如果获取到 targetSource 则去生成对应对象的代理对象
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    return null;
}

@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        // 获取 cache key
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

第一个方法 `postProcessBeforeInstantiation` 是在每个容器实例化每个 Bean 之前调用，调用栈大致如下：

容器调用 `refresh` 方法刷新，在其中调用 `beanFactory.preInstantiateSingletons` 方法去生成所有的被 Spring 管理的 Bean 对象，根据 Bean 定义信息首先调用 `getBean` 方法尝试从容器中获取 Bean，`getBean` 直接调用 `doGetBean` 获取，在其中调用 `getSingleton` 获取 Bean。在获取不到的情况使用 `createBean` 创建 Bean，在创建之前调用 `resolveBeforeInstantiation` 方法给 BeanProcessor 一个机会去返回目标对象的一个代理对象，其要实现 `InstantiationAwareBeanPostProcessor` 。

```java
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}

@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
}

@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```

允许每个 Bean 后置处理器在 Bean 实例化之前和之后调用相应的方法，第一个方法也就是在这个时刻调用的。

第二个方法是 `postProcessAfterInitialization` 在初始化之后调用。接着上面的方法，如果没有返回 Bean 实例的话，则会调用 `doCreateBean` 创建对象，在其中调用 `instanceWrapper.getWrappedInstance();`  去创建一个 Bean 包装实例，之后调用 `populateBean(beanName, mbd, instanceWrapper);` 为 Bean 设置相关的属性，之后调用 `initializeBean` 去调用 Bean 的初始化方法，和上面类似

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // 首先调用 Bean 的 Aware 方法为 Bean 设置一些相关的属性
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 遍历调用后置处理器的 postProcessBeforeInitialization 方法
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 调用初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 遍历调用后置处理器的 postProcessAfterInitialization 方法
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

在 `AspectJAwareAdvisorAutoProxyCreator` 的 `postProcessAfterInitialization` 中调用 `wrapIfNecessary` 包装 Bean，

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
	
    // Create proxy if we have advice.
    // 判断是否有 advisors，如果有才创建代理
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        // 经过上面的校验，如果能够到达此处，则开始创建包装对象代理
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 创建代理
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}

@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
    Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    // 获取 Bean 的advisors
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

// 创建代理
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        // 暴露 TargetClass，将 target class 放到标签中 
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    if (!proxyFactory.isProxyTargetClass()) {
        // 直接代理类
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            // 尝试代理接口
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 感觉 Bean 名称和特定的拦截器去生成 Advisor
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 代理工厂设置 Advisor
    proxyFactory.addAdvisors(advisors);
    // 代理工厂设置目标类
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    return proxyFactory.getProxy(getProxyClassLoader());
}

protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
    // 获取接口
    Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
    boolean hasReasonableProxyInterface = false;
    for (Class<?> ifc : targetInterfaces) {
        if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
            ifc.getMethods().length > 0) {
            hasReasonableProxyInterface = true;
            break;
        }
    }
    // 判断是否有可用的代理接口
    if (hasReasonableProxyInterface) {
        // Must allow for introductions; can't just set interfaces to the target's interfaces only.
        for (Class<?> ifc : targetInterfaces) {
            proxyFactory.addInterface(ifc);
        }
    }
    else {
        // 直接代理类
        proxyFactory.setProxyTargetClass(true);
    }
}

// 获取代理
public Object getProxy(@Nullable ClassLoader classLoader) {
    // 创建代理，如果是接口可以直接使用 JDK 的代理，否则使用 CGLIB 创建代理
    return createAopProxy().getProxy(classLoader);
}

// 使用 CGLIB 创建代理
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isTraceEnabled()) {
        logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
    }

    try {
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

        Class<?> proxySuperClass = rootClass;
        if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
            proxySuperClass = rootClass.getSuperclass();
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }

        // Validate the class, writing log messages as necessary.
        validateClassIfNecessary(proxySuperClass, classLoader);

        // Configure CGLIB Enhancer...
        Enhancer enhancer = createEnhancer();
        if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                enhancer.setUseCache(false);
            }
        }
        enhancer.setSuperclass(proxySuperClass);
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        // fixedInterceptorMap only populated at this point, after getCallbacks call above
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
            this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        // Generate the proxy class and create a proxy instance.
        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (CodeGenerationException | IllegalArgumentException ex) {
        throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                                     ": Common causes of this problem include using a final class or a non-visible class",
                                     ex);
    }
    catch (Throwable ex) {
        // TargetSource.getTarget() failed
        throw new AopConfigException("Unexpected AOP exception", ex);
    }
}
```

至此所有的两个主要方法就执行完毕了，也就生成了对应的代理对象了。

**下面看一下调用方法时候的处理流程：**

例如调用 `Message.sayHello` 方法，在获取到 Bean 的时候发现已经是代理之后的对象了 ，如下图所示:

![被代理的对象]( http://img.sangzhenya.com/Snipaste_2019-11-21_23-18-48.png )

首先是 `CglibAopProxy.DynamicAdvisedInterceptor` 中的 `intercept ` 方法

```java
@Override
@Nullable
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    TargetSource targetSource = this.advised.getTargetSource();
    try {
        if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
        // 获取被代理的类
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);
        // 从 advised 中获取目标方法的拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        // Check whether we only have one InvokerInterceptor: that is,
        // no real advice, but just reflective invocation of the target.
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            // We can skip creating a MethodInvocation: just invoke the target directly.
            // Note that the final invoker must be an InvokerInterceptor, so we know
            // it does nothing but a reflective operation on the target, and no hot
            // swapping or fancy proxying.
            // 如果没有任何拦截，则直接调用返回
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // We need to create a method invocation...
            // 创建 CglibMethodInvocation 并调用 proceed 处理
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        retVal = processReturnType(proxy, target, method, retVal);
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

上图中获取的拦截器链如下，其实就是定义的切面方法。

![拦截链]( http://img.sangzhenya.com/Snipaste_2019-11-23_12-17-05.png )

获取拦截器链的方法如下：
```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    // 生成 cachkey key
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    // 获取 cachkey
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        // 如果 cache 为空则去获取
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```

```java
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
    Advised config, Method method, @Nullable Class<?> targetClass) {

    // This is somewhat tricky... We have to process introductions first,
    // but we need to preserve order in the ultimate list.
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
    // 获取到类的 advisor
    Advisor[] advisors = config.getAdvisors();
    // 保存所有拦截器
    List<Object> interceptorList = new ArrayList<>(advisors.length);
    Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
    Boolean hasIntroductions = null;

    // 遍历增强器集合过滤生成 method 对应的 拦截器链
    for (Advisor advisor : advisors) {
        if (advisor instanceof PointcutAdvisor) {
            // Add it conditionally.
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                // 获取 Method Matcher 并进行 Match
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                boolean match;
                if (mm instanceof IntroductionAwareMethodMatcher) {
                    if (hasIntroductions == null) {
                        hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                    }
                    match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
                }
                else {
                    match = mm.matches(method, actualClass);
                }
                // 如果 match，则将 增强器转换成 MethodInterceptor
                if (match) {
                    MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                    if (mm.isRuntime()) {
                        // Creating a new object instance in the getInterceptors() method
                        // isn't a problem as we normally cache created chains.
                        for (MethodInterceptor interceptor : interceptors) {
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    }
                    else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
        else {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }

    return interceptorList;
}
```



```java
// 根据 Advisor 生成 MethodInterceptor
@Override
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<>(3);
    Advice advice = advisor.getAdvice();
    // 直接生成
    if (advice instanceof MethodInterceptor) {
        interceptors.add((MethodInterceptor) advice);
    }
    // 通过适配器生成
    for (AdvisorAdapter adapter : this.adapters) {
        if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
        }
    }
    if (interceptors.isEmpty()) {
        throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[0]);
}
```

得到拦截器链之后就可以根据拦截器链去执行方法了，下面是 `proceed` 方法的流程

```java
@Override
@Nullable
public Object proceed() throws Throwable {
    // We start with an index of -1 and increment early.
    // 递归调用，从 -1 开始递增
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    // 获取当前的拦截器
    Object interceptorOrInterceptionAdvice =
        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // Evaluate dynamic method matcher here: static part will already have
        // been evaluated and found to match.
        InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
        if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        else {
            // Dynamic matching failed.
            // Skip this interceptor and invoke the next in the chain.
            return proceed();
        }
    }
    else {
        // It's an interceptor, so we just invoke it: The pointcut will have
        // been evaluated statically before this object was constructed.
        // 调用拦截器的 invoke，递归调用
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

有 5 种 `Interceptor ` 如下所示：

```java
// MethodBeforeAdviceInterceptor
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
    return mi.proceed();
}
```

```java
// AspectJAfterAdvice
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        return mi.proceed();
    }
    finally {
        invokeAdviceMethod(getJoinPointMatch(), null, null);
    }
}
```

```java
// AfterReturningAdviceInterceptor
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    Object retVal = mi.proceed();
    this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
    return retVal;
}
```

```java
// AspectJAfterThrowingAdvice
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        return mi.proceed();
    }
    catch (Throwable ex) {
        if (shouldInvokeOnThrowing(ex)) {
            invokeAdviceMethod(getJoinPointMatch(), null, ex);
        }
        throw ex;
    }
}
```

```java
// AspectJAroundAdvice
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    if (!(mi instanceof ProxyMethodInvocation)) {
        throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
    }
    ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
    ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
    JoinPointMatch jpm = getJoinPointMatch(pmi);
    return invokeAdviceMethod(pjp, jpm, null, null);
}
```

