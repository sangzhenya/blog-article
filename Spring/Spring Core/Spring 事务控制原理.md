---
title: "Spring 事务控制原理"
tags: ["Spring", "源码"]
categories: ["Spring"]
date: "2019-01-17T09:00:00+08:00"
---

使用 Spring 注解的方式启用事务，首先要使用 `@EnableTransactionManagement` 注解开启全局事务，事务需要交给具体的 TransactionManager 进行管理，所以需要向容器中添加相应的 Bean，如下所示

```java
 @Bean
 public PlatformTransactionManager transactionManager() throws PropertyVetoException {
 return new DataSourceTransactionManager(dataSource());
 }
```

最后需要在具体需要添加事务的方法上加上 `@Transactional` 注解。

Spring 的事务隔离级别有以下几种：

1. `TransactionDefinition.ISOLATION_DEFAULT`（使用后端数据库默认的隔离级别）
2. `TransactionDefinition.ISOLATION_READ_UNCOMMITTED`
3. `TransactionDefinition.ISOLATION_READ_COMMITTED`
4. `TransactionDefinition.ISOLATION_REPEATABLE_READ`
5. `TransactionDefinition.ISOLATION_SERIALIZABLE`

Spring 的事务传播行为以下几种：

1. `TransactionDefinition.PROPAGATION_REQUIRED`： 有则加入，没有则新建事务
2. `TransactionDefinition.PROPAGATION_SUPPORTS`：有则加入，没有则以非事务模式运行
3. `TransactionDefinition.PROPAGATION_MANDATORY`： 有则加入，没有则抛出异常
4. `TransactionDefinition.PROPAGATION_REQUIRES_NEW`： 将当前事务挂起，新建一个事务
5. `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`：将当前事务挂起，以非事务运行
6. `TransactionDefinition.PROPAGATION_NEVER`：当前有事务则抛出异常，否则以非事务运行
7. `TransactionDefinition.PROPAGATION_NESTED` 嵌套事务



具体 Spring 实现事务的原理如下：

在 `@EnableTransactionManagement` 注解中引入了 `TransactionManagementConfigurationSelector` 类，该类如下：

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
        // 根据不同的模式导入不同的 config，默认是 PROXY 模式
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}

	private String determineTransactionAspectClass() {
		return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
				TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
				TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
	}

}
```

该类会为容器引入 `AutoProxyRegistrar` 和 `ProxyTransactionManagementConfiguration` 两个类。

其中 `AutoProxyRegistrar` 实现了 `ImportBeanDefinitionRegistrar` 接口，在 `registerBeanDefinitions()` 方法中为容器中注册 `InfrastructureAdvisorAutoProxyCreator` 类，该类是一个后置处理器，在对象创建以后将对象包装。返回一个代理对象（增强器），代理对象执行方法利用拦截器链进行调用。

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

`ProxyTransactionManagementConfiguration` 为容器中添加了一个 `BeanFactoryTransactionAttributeSourceAdvisor` 事务增强器。

```java
@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
    TransactionAttributeSource transactionAttributeSource,
    TransactionInterceptor transactionInterceptor) {
    BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
    advisor.setTransactionAttributeSource(transactionAttributeSource);
    advisor.setAdvice(transactionInterceptor);
    if (this.enableTx != null) {
        advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
    }
    return advisor;
}
@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public TransactionAttributeSource transactionAttributeSource() {
    return new AnnotationTransactionAttributeSource();
}

@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public TransactionInterceptor transactionInterceptor(
    TransactionAttributeSource transactionAttributeSource) {
    TransactionInterceptor interceptor = new TransactionInterceptor();
    interceptor.setTransactionAttributeSource(transactionAttributeSource);
    if (this.txManager != null) {
        interceptor.setTransactionManager(this.txManager);
    }
    return interceptor;
}
```

同时为容器中注入了 `TransactionAttributeSource` 事务属性类，为容器加入事务属性注解解析类，包含 Spring，JTA，EJB 用去处理事务的注解信息

```java
public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
    this.publicMethodsOnly = publicMethodsOnly;
    if (jta12Present || ejb3Present) {
        this.annotationParsers = new LinkedHashSet<>(4);
        this.annotationParsers.add(new SpringTransactionAnnotationParser());
        if (jta12Present) {
            this.annotationParsers.add(new JtaTransactionAnnotationParser());
        }
        if (ejb3Present) {
            this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
        }
    }
    else {
        this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());
    }
}
```

同时为容器注入了 `TransactionInterceptor` 事务拦截器：保存了 `transactionAttributeSource` 事务属性信息和 `txManager` 事务管理器。该尅本身实现了 `MethodInterceptor` 是一个方法拦截器，在 `invoke` 方法中调用 `invokeWithinTransaction` 以事务的方式运行方法。
```java
@Override
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {
    // Work out the target class: may be {@code null}.
    // The TransactionAttributeSource should be passed the target class
    // as well as the method, which may be from an interface.
    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // Adapt to TransactionAspectSupport's invokeWithinTransaction...
    return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}
```

`invokeWithinTransaction` 方法如下：

```java
@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
                                         final InvocationCallback invocation) throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    // 获取事务属性
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    // 获取 transactionManager
    final TransactionManager tm = determineTransactionManager(txAttr);

    if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager) {
        ReactiveTransactionSupport txSupport = this.transactionSupportCache.computeIfAbsent(method, key -> {
            if (KotlinDetector.isKotlinType(method.getDeclaringClass()) && KotlinDelegate.isSuspend(method)) {
                throw new TransactionUsageException(
                    "Unsupported annotated transaction on suspending function detected: " + method +
                    ". Use TransactionalOperator.transactional extensions instead.");
            }
            ReactiveAdapter adapter = this.reactiveAdapterRegistry.getAdapter(method.getReturnType());
            if (adapter == null) {
                throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type: " +
                                                method.getReturnType());
            }
            return new ReactiveTransactionSupport(adapter);
        });
        return txSupport.invokeWithinTransaction(
            method, targetClass, invocation, txAttr, (ReactiveTransactionManager) tm);
    }

    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // This is an around advice: Invoke the next interceptor in the chain.
            // This will normally result in a target object being invoked.
            // 运行方法
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // target invocation exception
           	// 出现异常后回滚，然后抛出异常
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            // 处理事务信息
            cleanupTransactionInfo(txInfo);
        }

        if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
            // Set rollback-only in case of Vavr failure matching our rollback rules...
            TransactionStatus status = txInfo.getTransactionStatus();
            if (status != null && txAttr != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
            }
        }
		// commit 改动
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }

    else {
        final ThrowableHolder throwableHolder = new ThrowableHolder();

        // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
        try {
            Object result = ((CallbackPreferringPlatformTransactionManager) ptm).execute(txAttr, status -> {
                // 开启事务
                TransactionInfo txInfo = prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
                try {
                    Object retVal = invocation.proceedWithInvocation();
                    if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                        // Set rollback-only in case of Vavr failure matching our rollback rules...
                        retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                    }
                    return retVal;
                }
                catch (Throwable ex) {
                    if (txAttr.rollbackOn(ex)) {
                        // A RuntimeException: will lead to a rollback.
                        if (ex instanceof RuntimeException) {
                            throw (RuntimeException) ex;
                        }
                        else {
                            throw new ThrowableHolderException(ex);
                        }
                    }
                    else {
                        // A normal return value: will lead to a commit.
                        throwableHolder.throwable = ex;
                        return null;
                    }
                }
                finally {
                    cleanupTransactionInfo(txInfo);
                }
            });

            // Check result state: It might indicate a Throwable to rethrow.
            if (throwableHolder.throwable != null) {
                throw throwableHolder.throwable;
            }
            return result;
        }
        catch (ThrowableHolderException ex) {
            throw ex.getCause();
        }
        catch (TransactionSystemException ex2) {
            if (throwableHolder.throwable != null) {
                logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
                ex2.initApplicationException(throwableHolder.throwable);
            }
            throw ex2;
        }
        catch (Throwable ex2) {
            if (throwableHolder.throwable != null) {
                logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
            }
            throw ex2;
        }
    }
}
```

获取 `TransactionManager ` 的方法如下：

```java
@Nullable
protected TransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
    // Do not attempt to lookup tx manager if no tx attributes are set
    if (txAttr == null || this.beanFactory == null) {
        return getTransactionManager();
    }

    String qualifier = txAttr.getQualifier();
    if (StringUtils.hasText(qualifier)) {
        return determineQualifiedTransactionManager(this.beanFactory, qualifier);
    }
    else if (StringUtils.hasText(this.transactionManagerBeanName)) {
        return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
    }
    else {
        TransactionManager defaultTransactionManager = getTransactionManager();
        if (defaultTransactionManager == null) {
            defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
            if (defaultTransactionManager == null) {
                defaultTransactionManager = this.beanFactory.getBean(TransactionManager.class);
                this.transactionManagerCache.putIfAbsent(
                    DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
            }
        }
        return defaultTransactionManager;
    }
}

private TransactionManager determineQualifiedTransactionManager(BeanFactory beanFactory, String qualifier) {
    TransactionManager txManager = this.transactionManagerCache.get(qualifier);
    if (txManager == null) {
        txManager = BeanFactoryAnnotationUtils.qualifiedBeanOfType(
            beanFactory, TransactionManager.class, qualifier);
        this.transactionManagerCache.putIfAbsent(qualifier, txManager);
    }
    return txManager;
}
```

失败的时候会使用事务管理器回滚
```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
                         "] after exception: " + ex);
        }
        // 这边还要判断一下是否标注了 rollbackon 标签，并 Exception 是否相同
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
        }
        else {
            // We don't roll back on this exception.
            // Will still roll back if TransactionStatus.isRollbackOnly() is true.
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                throw ex2;
            }
        }
    }
}
```

正常结束就调用事务管理器提交
```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
        }
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}
```



此外通过 `TransactionSynchronizationManager`  可以在事务的各个触发点做相应的操作，例如判断当前是否在事务中：

`TransactionSynchronizationManager.isSynchronizationActive()` 或者注册事务结束之后的事件：

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override
    public void afterCommit() {
    }
});
```

主要可以注册事件的点如下：

```java
public interface TransactionSynchronization extends Flushable {

	/** Completion status in case of proper commit. */
	int STATUS_COMMITTED = 0;

	/** Completion status in case of proper rollback. */
	int STATUS_ROLLED_BACK = 1;

	/** Completion status in case of heuristic mixed completion or system errors. */
	int STATUS_UNKNOWN = 2;

	default void suspend() {
	}
	default void resume() {
	}
	@Override
	default void flush() {
	}
	default void beforeCommit(boolean readOnly) {
	}
	default void beforeCompletion() {
	}
	default void afterCommit() {
	}
	default void afterCompletion(int status) {
	}

}

```

这些事件会在 `TransactionManager` 执行到相应的点的时候触发。