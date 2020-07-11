---
title: "Spring BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor"
tags: ["Spring", "源码"]
categories: ["Spring"]
date: "2019-01-15T09:00:00+08:00"
---

`BeanFactoryPostProcessor` BeanFactory 的后置处理器；在 BeanFactory 标准初始化之后调用，所有 Bean 定义已经保存加载到 beanFactory 中但是 bean 的实例还未创建，可以操作 beanFactory 进行定制。为容器中添加一个自定义的 `BeanFactoryPostProcessoer` 如下：

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    private Log log = LogFactory.getLog(MyBeanFactoryPostProcessor.class);

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        int count = beanFactory.getBeanDefinitionCount();
        log.info("Count::" + count);
        String[] definitionNames = beanFactory.getBeanDefinitionNames();
        log.info("DefinitionNames::" + Arrays.toString(definitionNames));
    }
}
```

`BeanDefinitionRegistryPostProcessor` Bean 定义注册中心的后置处理器，运行修改 bean 定义注册中心。和 `BeanFactoryPostProcessor` 调用的位置之前， Bean 定义信息加载到 BeanFactory 之前同样也是在 Bean 的实例创建之前，但在可以自定义一个如下所示：

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {}
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
//        RootBeanDefinition beanDefinition = new RootBeanDefinition(MessageService.class);
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(MessageService.class).getBeanDefinition();
        registry.registerBeanDefinition("cMessageService", beanDefinition);
    }
}
```



然后查看其调用栈：

首先初始化容器，在这个过程中调用 `refresh`  方法刷新容器，刷新容器其实就是 Spring 启动的主要过程。在 `refresh`方法中调用 `invokeBeanFactoryPostProcessors` 方法去 invoke BeanFactory 的后置处理器去执行相应的操作，调用 `PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors` 方法去组织调用相应的 BeanFactoryPostProcesser。 方法如下：

```java
// 方法传入的是 beanFactory 和 在上一步获取到的所有的 BeanFactoryPostProcessor
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    
    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    Set<String> processedBeans = new HashSet<>();

    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 从传入的 beanFactoryPostProcessors 分类收集 BeanFactoryPostProcessor
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                    (BeanDefinitionRegistryPostProcessor) postProcessor;
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // 找到并执行实现了 PriorityOrdered  的 BeanDefinitionRegistryPostProcessor
        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
        String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 找到并执行实现了 Ordered  的 BeanDefinitionRegistryPostProcessor
        // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 找到并执行实现了剩下的 BeanDefinitionRegistryPostProcessor
        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // 因为 BeanDefinitionRegistryPostProcessor 同时又是 BeanFactoryPostProcessor
        // 所以在调用了其 postProcessBeanDefinitionRegistry 之后需要调用其继承来的
        // postProcessBeanFactory 方法
        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // 这里找到所有实现 BeanFactoryPostProcessor 接口的 Bean 的名称
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // 先分别放到三个不同的优先级的列表中
    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        // 如果再上面已经处理过了则跳过
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 首先调用实现了 PriorityOrdered 的 BeanFactoryPostProcessor
    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // 然后调用实现了 Ordered 的 BeanFactoryPostProcessor
    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);
	
    // 最后调用剩余的 BeanFactoryPostProcessor
    // Finally, invoke all other BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    beanFactory.clearMetadataCache();
}
```

调用的方法也很简单：

```java
private static void invokeBeanDefinitionRegistryPostProcessors(
    Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {
	// 遍历所有的 BeanDefinitionRegistryPostProcessor 并调用其 postProcessBeanDefinitionRegistry 即可
    for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanDefinitionRegistry(registry);
    }
}
private static void invokeBeanFactoryPostProcessors(
    Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
	// 遍历所有的 BeanFactoryPostProcessor 并调用其 postProcessBeanFactory 即可
    for (BeanFactoryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanFactory(beanFactory);
    }
}
```

