## BeanPostProcessor 原理

BeanPostProcessor 是 Spring 提供的一个钩子，可以对新创建的 Bean 进行一些定制。例如检查标记接口和包装成为代理对象。容器能够自动检测到实现了该接口的 Bean，并应用在其他的 Bean 创建过程之中。Bean Factory 允许编程方式注册 BeanPostProcessor 用于所有通过该 BeanFactory 创建的 Bean 上。

该接口有两个方法，`postProcessBeforeInitialization` 和 `postProcessAfterInitialization`，参数值均是新创建的 Bean 实例和该实例的 Name。返回值是放到容器中被使用的 Bean，可以是原始的 Bean 实例，也可以是该 Bean 的一个包装对象。

一般来讲，通过检测标记接口给 Bean 添加属性的使用 `postProcessBeforeInitialization` 方法，而包装 Bean 返回代理对象的使用 `postProcessAfterInitialization`。这也是 BeanPostProcessor 两个主要用途。分别可以参考： ApplicationContextAwareProcessor 和 AnnotationAwareAspectJAutoProxyCreator

BeanPostProcessor 可以指定 Bean 的初始化之前和初始化之后调用的方法，一个简单的例子如下：


```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization" + bean + "::" + beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization" + bean + "::" + beanName);
        return bean;
    }
}
```

调用过程栈如下：
1. 首先启动容器, 新建容器默认调用其构造方法：
```java
@Test
void testAOP() {
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig.class);
    applicationContext.close();
}
```
2. 启动容器调用 refresh 方法, 刷新容器。
```java
@Override
	public void refresh() throws BeansException, IllegalStateException {
	/***省略部分代码***/
    // Instantiate all remaining (non-lazy-init) singletons.
	finishBeanFactoryInitialization(beanFactory);	    
	/***省略部分代码***/
}

```
3. refresh 方法中调用 finishBeanFactoryInitialization 初始化所有的非懒加载的单实例 Bean.
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    /***省略部分代码***/
    // Instantiate all remaining (non-lazy-init) singletons.
	beanFactory.preInstantiateSingletons();
}
```
4. finishBeanFactoryInitialization 方法中调用 preInstantiateSingletons 去初始化所有剩余的非拦加载的单实例 Bean。
```java
public void preInstantiateSingletons() throws BeansException {
    getBean(beanName);
}
```
5. preInstantiateSingletons 调用 getBean 方法尝试去获取 Bean
```java
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```
6. getBean 方法中直接调用 doGetBean 方法区获取 Bean
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	sharedInstance = getSingleton(beanName, () -> {
    	try {
    		return createBean(beanName, mbd, args);
    	}
    	catch (BeansException ex) {
    		// Explicitly remove instance from singleton cache: It might have been put there
    		// eagerly by the creation process, to allow for circular reference resolution.
    		// Also remove any beans that received a temporary reference to the bean.
    		destroySingleton(beanName);
    		throw ex;
    	}
    });	    
}
```
7. doGetBean 中调用 getSingleton 方法区获取单实例的 Bean。
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    singletonObject = singletonFactory.getObject();
}
```
8. 获取不到则 getSingleton 调用该函数传入的 ObjectFactory 的 get 方法创建对象，从该函数的调用的地方该方法就是一个 createBean 方法。即调用 createBean 方法区创建。
```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
	Object beanInstance = doCreateBean(beanName, mbdToUse, args);		    
}
```
9. createBean 方法中调用 doCreateBean 去真正的执行创建的操作。
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
		throws BeanCreationException {
	populateBean(beanName, mbd, instanceWrapper);
	exposedObject = initializeBean(beanName, exposedObject, mbd);	    
}
```
10. doCreateBean 方法中调用了 populateBean 为 新创建的 Bean 属性赋值，调用 initializeBean 方法区初始化 Bean。
```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
	    // 应用 BeanPostProcessor 的 postProcessBeforeInitialization 方法
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
	    // 执行初始化方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
        // 应用 BeanPostProcessor 的 applyBeanPostProcessorsAfterInitialization 方法
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}

	return wrappedBean;
}
```
11. initializeBean 方法中分别 postProcessBeforeInitialization 和 applyBeanPostProcessorsAfterInitialization 执行 BeanPostProcessor 中的 postProcessBeforeInitialization 和 postProcessAfterInitialization 方法。
```java 
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	// 获取当前容器中所有的 BeanPostProcessor, 逐个调用 postProcessBeforeInitialization 方法
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessBeforeInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```
```java 
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	// 获取当前容器中所有的 BeanPostProcessor, 逐个调用 postProcessAfterInitialization 方法
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