## Dubbo 原理

参考：[Dubbo 开发者指南](http://dubbo.apache.org/zh-cn/docs/dev/design.html)

Dubbo 整体框架设计如下：

![框架设计](http://img.programya.com/20200104110704.png)

图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。

主要包含 config 层，proxy 服务代理层，registry 注册中心层，cluster 路由层，monitor 监控层，protocol 远程调用层，exchange 信息交换层，transport 网络传输层，serialize 数据序列化层。

### 启动流程

一个简单的 Dubbo Provider 的配置如下：

```yaml
dubbo:
  application:
    name: dubbo-provider
  registry:
    address: zookeeper://192.168.0.111:2181
  protocol:
    name: dubbo
    port: 20080
  scan:
    base-packages: com.xinyue.dubbo.provider.service
```

对应的是 `DubboConfigurationProperties` 配置类。

在 `DubboAutoConfiguration` 定义如下，为容器中引入了一些 Bean。

```java
@ConditionalOnProperty(prefix = DUBBO_PREFIX, name = "enabled", matchIfMissing = true)
@Configuration
// 自动配置后调用
@AutoConfigureAfter(DubboRelaxedBindingAutoConfiguration.class)
// 启用属性配置
@EnableConfigurationProperties(DubboConfigurationProperties.class)
public class DubboAutoConfiguration {
```

在 `dubbo-spring-boot-starter` 包中引入了 `dubbo-spring-boot-autoconfigure` 依赖，其又为引入了 `dubbo-spring-boot-autoconfigure-compatible` 依赖，在该包中的 `META-INFO` 下的 `spring.factories` 文件中定义了 AutoConfig，ApplicationListener 等类如下：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.apache.dubbo.spring.boot.autoconfigure.DubboAutoConfiguration,\
org.apache.dubbo.spring.boot.autoconfigure.DubboRelaxedBindingAutoConfiguration
org.springframework.context.ApplicationListener=\
org.apache.dubbo.spring.boot.context.event.OverrideDubboConfigApplicationListener,\
org.apache.dubbo.spring.boot.context.event.WelcomeLogoApplicationListener,\
org.apache.dubbo.spring.boot.context.event.AwaitingNonWebApplicationListener
org.springframework.boot.env.EnvironmentPostProcessor=\
org.apache.dubbo.spring.boot.env.DubboDefaultPropertiesEnvironmentPostProcessor
org.springframework.context.ApplicationContextInitializer=\
org.apache.dubbo.spring.boot.context.DubboApplicationContextInitializer
```

首先在 Spring Boot 启动的时候会调用 `SpringApplication.prepareEnvironment` 方法中会调用 `SpringApplicationRunListeners` 的 `environmentPrepared` 方法。在其中会使用初始化的event 派发器派发 `ApplicationEnvironmentPreparedEvent` 事件。获取所有的 `ApplicationListener` 逐个调用 `onApplicationEvent` 方法。在 `ConfigFileApplicationListener` 类中的 `onApplicationEvent` 中中会逐个调用 `EnvironmentPostProcessor` 的`postProcessEnvironment` 方法。在 `DubboDefaultPropertiesEnvironmentPostProcessor` 中会从环境变量中取出部分属性（例如 `dubbo.application.name`） 作为 dubbo 的属性放到环境变量中。

```java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
   MutablePropertySources propertySources = environment.getPropertySources();
   // 创建 Default 属性e
   Map<String, Object> defaultProperties = createDefaultProperties(environment);
   if (!CollectionUtils.isEmpty(defaultProperties)) {
     addOrReplace(propertySources, defaultProperties);
   }
}
private Map<String, Object> createDefaultProperties(ConfigurableEnvironment environment) {
  Map<String, Object> defaultProperties = new HashMap<String, Object>();
  setDubboApplicationNameProperty(environment, defaultProperties);
  setDubboConfigMultipleProperty(defaultProperties);
  setDubboApplicationQosEnableProperty(defaultProperties);
  setAllowBeanDefinitionOverriding(defaultProperties);
  return defaultProperties;
}
```

之后调用到 `WelcomeLogoApplicationListener` 的 `onApplicationEvent` 方法，主要是为了打印 Dubbo 的  Banner 信息。

之后调用到 `OverrideDubboConfigApplicationListener` 的 `onApplicationEvent` 方法，主要是为了拿到所有的 Dubbo 的属性信息然后放到 `ConfigUtils` 的 `PROPERTIES` 上。

在准备好环境之后开始准备刷新容器，调用 `SpringApplication` 的 `prepareContext` 方法。在其中会调用 `applyInitializers` 方法应用初始化，在其中会获取到所有的  `ApplicationContextInitializer` 的类调用其 `initialize` 方法。所以这里会调用到 `DubboApplicationContextInitializer` 的 `initialize` 方法主要为容器中添加两个 `BeanFactoryPostProcessor` 类：`OverrideBeanDefinitionRegistryPostProcessor`， `DubboConfigBeanDefinitionConflictProcessor`。

```java
// DubboApplicationContextInitializer
public void initialize(ConfigurableApplicationContext applicationContext) {
  overrideBeanDefinitions(applicationContext);
}

private void overrideBeanDefinitions(ConfigurableApplicationContext applicationContext) {
  applicationContext.addBeanFactoryPostProcessor(new OverrideBeanDefinitionRegistryPostProcessor());
  applicationContext.addBeanFactoryPostProcessor(new DubboConfigBeanDefinitionConflictProcessor());
}
```

然后调用  `refreshContext` 去刷新容器，在刷新容器的过程中有一部调用 `invokeBeanFactoryPostProcessors` 方法去调用 Bean Factory Processor 为容器中注册 Bean 信息。`OverrideBeanDefinitionRegistryPostProcessor` 的 `postProcessBeanDefinitionRegistry` 方法中，为容器中注册了名称为 `namePropertyDefaultValueDubboConfigBeanCustomizer` 的 `DubboConfigBeanCustomizer` 类。

在 `ConfigurationClassPostProcessor` 中 `postProcessBeanDefinitionRegistry`方法中会调用 `processConfigBeanDefinitions` 方法根据 `registry` 信息从配置类的加载 Bean 的定义信息。在 `DubboConfigBindingsRegistrar` 类的 `registerBeanDefinitions` 方法中

```java
// DubboConfigBindingsRegistrar
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  AnnotationAttributes attributes = AnnotationAttributes.fromMap(
    importingClassMetadata.getAnnotationAttributes(EnableDubboConfigBindings.class.getName()));
  // 获取注解并逐个注册到容器中
  AnnotationAttributes[] annotationAttributes = attributes.getAnnotationArray("value");
  DubboConfigBindingRegistrar registrar = new DubboConfigBindingRegistrar();
  registrar.setEnvironment(environment);
  for (AnnotationAttributes element : annotationAttributes) {
    registrar.registerBeanDefinitions(element, registry);
  }
}
```

```java
// DubboConfigBindingRegistrar
protected void registerBeanDefinitions(AnnotationAttributes attributes, BeanDefinitionRegistry registry) {
  // 获取属性的前缀  例如 dubbo.applications
  String prefix = environment.resolvePlaceholders(attributes.getString("prefix"));
  // 找到对应的 type 例如 org.apache.dubbo.config.ApplicationConfig
  Class<? extends AbstractConfig> configClass = attributes.getClass("type");
  boolean multiple = attributes.getBoolean("multiple");
  registerDubboConfigBeans(prefix, configClass, multiple, registry);
}
private void registerDubboConfigBeans(String prefix, Class<? extends AbstractConfig> configClass,
                                      boolean multiple, BeanDefinitionRegistry registry) {
  // 从环境中获取配置，如果没有相关的配置则直接返回
  Map<String, Object> properties = getPrefixedProperties(environment.getPropertySources(), prefix);
  if (CollectionUtils.isEmpty(properties)) {
    if (log.isDebugEnabled()) {
      log.debug("There is no property for binding to dubbo config class [" + configClass.getName()
                + "] within prefix [" + prefix + "]");
    }
    return;
  }
  // 否则 则去获取到 Bean Name 并注册 Dubbo Config 信息
  Set<String> beanNames = multiple ? resolveMultipleBeanNames(properties) :
  Collections.singleton(resolveSingleBeanName(properties, configClass, registry));
  for (String beanName : beanNames) {
    // 注册 Config Bean
    registerDubboConfigBean(beanName, configClass, registry);
    // 注册 Config Bean 绑定的 BeanPostProcessor
    registerDubboConfigBindingBeanPostProcessor(prefix, beanName, multiple, registry);
  }
  registerDubboConfigBeanCustomizers(registry);
}
private void registerDubboConfigBean(String beanName, Class<? extends AbstractConfig> configClass,
                                         BeanDefinitionRegistry registry) {
  // 根据 Config 的 class 获取对应的 builder
  BeanDefinitionBuilder builder = rootBeanDefinition(configClass);
  AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
  // 注册 bean 的 定义信息
  registry.registerBeanDefinition(beanName, beanDefinition);
  if (log.isInfoEnabled()) {
    log.info("The dubbo config bean definition [name : " + beanName + ", class : " + configClass.getName() +
             "] has been registered.");
  }
}
private void registerDubboConfigBindingBeanPostProcessor(String prefix, String beanName, boolean multiple,
                                                             BeanDefinitionRegistry registry) {
  Class<?> processorClass = DubboConfigBindingBeanPostProcessor.class;
  BeanDefinitionBuilder builder = rootBeanDefinition(processorClass);
  String actualPrefix = multiple ? buildPrefix(prefix) + beanName : prefix;
  builder.addConstructorArgValue(actualPrefix).addConstructorArgValue(beanName);
  AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
  beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
  registerWithGeneratedName(beanDefinition, registry);
  if (log.isInfoEnabled()) {
    log.info("The BeanPostProcessor bean definition [" + processorClass.getName()
             + "] for dubbo config bean [name : " + beanName + "] has been registered.");
  }
}
private void registerDubboConfigBeanCustomizers(BeanDefinitionRegistry registry) {
  // 注册 namePropertyDefaultValueDubboConfigBeanCustomizer 的 Bean
  registerInfrastructureBean(registry, BEAN_NAME, NamePropertyDefaultValueDubboConfigBeanCustomizer.class);
}
```

之后会初始化 `BeanDefinitionRegistryPostProcessor` ，其中会 `DubboAutoConfiguration ` 中获取 `ServiceAnnotationBeanPostProcessor` 并指定需要扫描的包。

```java
@ConditionalOnProperty(prefix = DUBBO_SCAN_PREFIX, name = BASE_PACKAGES_PROPERTY_NAME)
@ConditionalOnBean(name = BASE_PACKAGES_PROPERTY_RESOLVER_BEAN_NAME)
@Bean
public ServiceAnnotationBeanPostProcessor serviceAnnotationBeanPostProcessor(
  @Qualifier(BASE_PACKAGES_PROPERTY_RESOLVER_BEAN_NAME) PropertyResolver propertyResolver) {
  Set<String> packagesToScan = propertyResolver.getProperty(BASE_PACKAGES_PROPERTY_NAME, Set.class, emptySet());
  return new ServiceAnnotationBeanPostProcessor(packagesToScan);
}
```

在完成 `BeanDefinitionRegistryPostProcessor` 的创建之后调用 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法，在 `DubboConfigBeanDefinitionConflictProcessor` 方法中处理 Application Config Bean，

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    resolveUniqueApplicationConfigBean(registry, beanFactory);
}
```

此后在 `registerBeanPostProcessors` 方法中会将 `DubboAutoConfiguration` 中定义的 `ReferenceAnnotationBeanPostProcessor` 加入到容器中。

```java
 @ConditionalOnMissingBean
@Bean(name = ReferenceAnnotationBeanPostProcessor.BEAN_NAME)
public ReferenceAnnotationBeanPostProcessor referenceAnnotationBeanPostProcessor() {
  return new ReferenceAnnotationBeanPostProcessor();
}
```

最后在 `finishBeanFactoryInitialization` 方法中会创建处理 Dubbo 需要的几个 Config Bean。在创建的时候会调用 `DubboConfigBindingBeanPostProcessor` 的 `postProcessBeforeInitialization` 方法使用自己定义的配置信息配置相关的 Config 类。

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  if (this.beanName.equals(beanName) && bean instanceof AbstractConfig) {
    AbstractConfig dubboConfig = (AbstractConfig) bean;
    // 绑定配置
    bind(prefix, dubboConfig);
    // 定制 Config
    customize(beanName, dubboConfig);
  }
  return bean;
}

private void customize(String beanName, AbstractConfig dubboConfig) {
  // 逐个调用 Customizer 去定制
  for (DubboConfigBeanCustomizer customizer : configBeanCustomizers) {
    customizer.customize(beanName, dubboConfig);
  }
}

```



### 服务暴露

在准备好 Config 之后，就可以暴露出服务了。在容器刷新的最后一步你 `finishRefresh` 方法中会将刷新完成时间发布出去。`ServiceBean` 为监听这个事件并执行 `onApplicationEvent` 方法。 在该方法中会 export 服务。

```java
// ServiceBean
public void onApplicationEvent(ContextRefreshedEvent event) {
  if (!isExported() && !isUnexported()) {
    if (logger.isInfoEnabled()) {
      logger.info("The service ready on spring started. service: " + getInterface());
    }
    export();
  }
}
public void export() {
  // 调用其父类 ServiceConfig 的方法
  super.export();
  // Publish ServiceBeanExportedEvent
  publishExportEvent();
}
```

```java
// ServiceConfig
public synchronized void export() {
  checkAndUpdateSubConfigs();
  if (!shouldExport()) {
    return;
  }
  // 判断是否应该 delay 如果应该的话则设置 schedule
  if (shouldDelay()) {
    DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
  } else {
    // 否则开始 export
    doExport();
  }
}
// 检查和更笨 Config
public void checkAndUpdateSubConfigs() {
  // 使用全局的配置补全 Config
  completeCompoundConfigs();
  // 启动配置中心：从 ConfigManager 获取 configCenter
  startConfigCenter();
  // Check 默认值
  checkDefault();
  // Check 协议信息
  checkProtocol();
  // Check Application
  checkApplication();
  // if protocol is not injvm checkRegistry
  if (!isOnlyInJvm()) {
    // check 注册信息
    checkRegistry();
  }
  // 刷新
  this.refresh();
  // Check 数据信息
  checkMetadataReport();

  if (StringUtils.isEmpty(interfaceName)) {
    throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
  }

  // 是否是泛型 Service
  if (ref instanceof GenericService) {
    interfaceClass = GenericService.class;
    if (StringUtils.isEmpty(generic)) {
      generic = Boolean.TRUE.toString();
    }
  } else {
    try {
      // 获取对应的 interface class
      interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                                     .getContextClassLoader());
    } catch (ClassNotFoundException e) {
      throw new IllegalStateException(e.getMessage(), e);
    }
    // check interface 和 method
    checkInterfaceAndMethods(interfaceClass, methods);
    checkRef();
    generic = Boolean.FALSE.toString();
  }
  // 判断 local 信息
  if (local != null) {
    if (Boolean.TRUE.toString().equals(local)) {
      local = interfaceName + "Local";
    }
    Class<?> localClass;
    try {
      localClass = ClassUtils.forNameWithThreadContextClassLoader(local);
    } catch (ClassNotFoundException e) {
      throw new IllegalStateException(e.getMessage(), e);
    }
    if (!interfaceClass.isAssignableFrom(localClass)) {
      throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
    }
  }
  // 判断存根信息
  if (stub != null) {
    if (Boolean.TRUE.toString().equals(stub)) {
      stub = interfaceName + "Stub";
    }
    Class<?> stubClass;
    try {
      stubClass = ClassUtils.forNameWithThreadContextClassLoader(stub);
    } catch (ClassNotFoundException e) {
      throw new IllegalStateException(e.getMessage(), e);
    }
    if (!interfaceClass.isAssignableFrom(stubClass)) {
      throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
    }
  }
  // Legitimacy check of stub, note that: the local will deprecated
  checkStubAndLocal(interfaceClass);
  // Legitimacy check and setup of local simulated operations.
  checkMock(interfaceClass);
}
void startConfigCenter() {
  // 当前的 configCenter 如果为空的话则去从 ConfigManager 中获取 Config
  if (configCenter == null) {
    ConfigManager.getInstance().getConfigCenter().ifPresent(cc -> this.configCenter = cc);
  }
  if (this.configCenter != null) {
    // 刷新 configCenter
    this.configCenter.refresh();
    prepareEnvironment();
  }
  // 调用 ConfigManager 的刷新方法
  ConfigManager.getInstance().refreshAll();
}
// 刷新 config
public void refresh() {
  try {
    CompositeConfiguration compositeConfiguration = Environment.getInstance().getConfiguration(getPrefix(), getId());
    Configuration config = new ConfigConfigurationAdapter(this);
    if (Environment.getInstance().isConfigCenterFirst()) {
      // The sequence would be: SystemConfiguration -> AppExternalConfiguration -> ExternalConfiguration -> AbstractConfig -> PropertiesConfiguration
      compositeConfiguration.addConfiguration(4, config);
    } else {
      // The sequence would be: SystemConfiguration -> AbstractConfig -> AppExternalConfiguration -> ExternalConfiguration -> PropertiesConfiguration
      compositeConfiguration.addConfiguration(2, config);
    }

    // loop methods, get override value and set the new value back to method
    Method[] methods = getClass().getMethods();
    for (Method method : methods) {
      if (MethodUtils.isSetter(method)) {
        try {
          String value = StringUtils.trim(compositeConfiguration.getString(extractPropertyName(getClass(), method)));
          // isTypeMatch() is called to avoid duplicate and incorrect update, for example, we have two 'setGeneric' methods in ReferenceConfig.
          if (StringUtils.isNotEmpty(value) && ClassUtils.isTypeMatch(method.getParameterTypes()[0], value)) {
            method.invoke(this, ClassUtils.convertPrimitive(method.getParameterTypes()[0], value));
          }
        } catch (NoSuchMethodException e) {
          logger.info("Failed to override the property " + method.getName() + " in " +
                      this.getClass().getSimpleName() +
                      ", please make sure every property has getter/setter method provided.");
        }
      } else if (isParametersSetter(method)) {
        String value = StringUtils.trim(compositeConfiguration.getString(extractPropertyName(getClass(), method)));
        if (StringUtils.isNotEmpty(value)) {
          Map<String, String> map = invokeGetParameters(getClass(), this);
          map = map == null ? new HashMap<>() : map;
          map.putAll(convert(StringUtils.parseParameters(value), ""));
          invokeSetParameters(getClass(), this, map);
        }
      }
    }
  } catch (Exception e) {
    logger.error("Failed to override ", e);
  }
}
// 做 export
protected synchronized void doExport() {
  if (unexported) {
    throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
  }
  if (exported) {
    return;
  }
  exported = true;
	// 获取接口名称
  if (StringUtils.isEmpty(path)) {
    path = interfaceName;
  }
  // 执行 Export
  doExportUrls();
}
// 执行 export urls
private void doExportUrls() {
  // 获取中心的 urls
  List<URL> registryURLs = loadRegistries(true);
  for (ProtocolConfig protocolConfig : protocols) {
    // 生成 pathkey 和 provider model
    String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
    ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);
    ApplicationModel.initProviderModel(pathKey, providerModel);
    doExportUrlsFor1Protocol(protocolConfig, registryURLs);
  }
}
// 暴露 url
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
  String name = protocolConfig.getName();
  if (StringUtils.isEmpty(name)) {
    name = DUBBO;
  }

  // 元数据的 map
  Map<String, String> map = new HashMap<String, String>();
  map.put(SIDE_KEY, PROVIDER_SIDE);

  appendRuntimeParameters(map);
  appendParameters(map, metrics);
  appendParameters(map, application);
  appendParameters(map, module);
  // remove 'default.' prefix for configs from ProviderConfig
  // appendParameters(map, provider, Constants.DEFAULT_KEY);
  appendParameters(map, provider);
  appendParameters(map, protocolConfig);
  appendParameters(map, this);
  if (CollectionUtils.isNotEmpty(methods)) {
    for (MethodConfig method : methods) {
      appendParameters(map, method, method.getName());
      String retryKey = method.getName() + ".retry";
      if (map.containsKey(retryKey)) {
        String retryValue = map.remove(retryKey);
        if (Boolean.FALSE.toString().equals(retryValue)) {
          map.put(method.getName() + ".retries", "0");
        }
      }
      List<ArgumentConfig> arguments = method.getArguments();
      if (CollectionUtils.isNotEmpty(arguments)) {
        for (ArgumentConfig argument : arguments) {
          // convert argument type
          if (argument.getType() != null && argument.getType().length() > 0) {
            Method[] methods = interfaceClass.getMethods();
            // visit all methods
            if (methods != null && methods.length > 0) {
              for (int i = 0; i < methods.length; i++) {
                String methodName = methods[i].getName();
                // target the method, and get its signature
                if (methodName.equals(method.getName())) {
                  Class<?>[] argtypes = methods[i].getParameterTypes();
                  // one callback in the method
                  if (argument.getIndex() != -1) {
                    if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                      appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                      throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                    }
                  } else {
                    // multiple callbacks in the method
                    for (int j = 0; j < argtypes.length; j++) {
                      Class<?> argclazz = argtypes[j];
                      if (argclazz.getName().equals(argument.getType())) {
                        appendParameters(map, argument, method.getName() + "." + j);
                        if (argument.getIndex() != -1 && argument.getIndex() != j) {
                          throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                        }
                      }
                    }
                  }
                }
              }
            }
          } else if (argument.getIndex() != -1) {
            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
          } else {
            throw new IllegalArgumentException("Argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
          }

        }
      }
    } // end of methods for
  }

  if (ProtocolUtils.isGeneric(generic)) {
    map.put(GENERIC_KEY, generic);
    map.put(METHODS_KEY, ANY_VALUE);
  } else {
    String revision = Version.getVersion(interfaceClass, version);
    if (revision != null && revision.length() > 0) {
      map.put(REVISION_KEY, revision);
    }

    String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
    if (methods.length == 0) {
      logger.warn("No method found in service interface " + interfaceClass.getName());
      map.put(METHODS_KEY, ANY_VALUE);
    } else {
      map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
    }
  }
  if (!ConfigUtils.isEmpty(token)) {
    if (ConfigUtils.isDefault(token)) {
      map.put(TOKEN_KEY, UUID.randomUUID().toString());
    } else {
      map.put(TOKEN_KEY, token);
    }
  }
  // export service
  String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
  Integer port = this.findConfigedPorts(protocolConfig, name, map);
  URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

  if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
      .hasExtension(url.getProtocol())) {
    url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
      .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
  }

  String scope = url.getParameter(SCOPE_KEY);
  // don't export when none is configured
  if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

    // export to local if the config is not remote (export to remote only when config is remote)
    if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
      exportLocal(url);
    }
    // export to remote if the config is not local (export to local only when config is local)
    if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
      if (CollectionUtils.isNotEmpty(registryURLs)) {
        for (URL registryURL : registryURLs) {
          //if protocol is only injvm ,not register
          if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
            continue;
          }
          // 获取 url
          url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
          // 获取 monitor URL
          URL monitorUrl = loadMonitor(registryURL);
          if (monitorUrl != null) {
            url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
          }
          if (logger.isInfoEnabled()) {
            if (url.getParameter(REGISTER_KEY, true)) {
              logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
            } else {
              logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
          }

          // For providers, this is used to enable custom proxy to generate invoker
          String proxy = url.getParameter(PROXY_KEY);
          if (StringUtils.isNotEmpty(proxy)) {
            registryURL = registryURL.addParameter(PROXY_KEY, proxy);
          }

          // 获取 invoker
          Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
          // invoker 包装
          DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
          // export 代理
          Exporter<?> exporter = protocol.export(wrapperInvoker);
          // 添加到 exporter 中
          exporters.add(exporter);
        }
      } else {
        if (logger.isInfoEnabled()) {
          logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
        }
        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

        Exporter<?> exporter = protocol.export(wrapperInvoker);
        exporters.add(exporter);
      }
      /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
      MetadataReportService metadataReportService = null;
      if ((metadataReportService = getMetadataReportService()) != null) {
        // 发布 provider 的 metadata
        metadataReportService.publishProvider(url);
      }
    }
  }
  this.urls.add(url);
}
```

```java
// ConfigManager 
public void refreshAll() {
  // refresh all configs here,
  getApplication().ifPresent(ApplicationConfig::refresh);
  getMonitor().ifPresent(MonitorConfig::refresh);
  getModule().ifPresent(ModuleConfig::refresh);

  getProtocols().values().forEach(ProtocolConfig::refresh);
  getRegistries().values().forEach(RegistryConfig::refresh);
  getProviders().values().forEach(ProviderConfig::refresh);
  getConsumers().values().forEach(ConsumerConfig::refresh);
}
```

```java
// AbstractInterfaceConfig
protected void checkRegistry() {
  // 加载注册的  Registry 的信息
  loadRegistriesFromBackwardConfig();
  // 获取外部配置中的 Registry 的信息
  convertRegistryIdsToRegistries();
  // 遍历 registries 并逐个判断是否是合法的
  for (RegistryConfig registryConfig : registries) {
    if (!registryConfig.isValid()) {
      throw new IllegalStateException("No registry config found or it's not a valid config! " +
                                      "The registry config is: " + registryConfig);
    }
  }
  // 使用 registries 作为默认的 config center
  useRegistryForConfigIfNecessary();
}
private void useRegistryForConfigIfNecessary() {
  // 遍历 registries
  registries.stream().filter(RegistryConfig::isZookeeperProtocol).findFirst().ifPresent(rc -> {
    // we use the loading status of DynamicConfiguration to decide whether ConfigCenter has been initiated.
    Environment.getInstance().getDynamicConfiguration().orElseGet(() -> {
      ConfigManager configManager = ConfigManager.getInstance();
      ConfigCenterConfig cc = configManager.getConfigCenter().orElse(new ConfigCenterConfig());
      if (cc.getParameters() == null) {
        cc.setParameters(new HashMap<>());
      }
      if (rc.getParameters() != null) {
        cc.getParameters().putAll(rc.getParameters());
      }
      // 设置 参数/协议/地址/用户名/密码等信息
      cc.getParameters().put(org.apache.dubbo.remoting.Constants.CLIENT_KEY,rc.getClient());
      cc.setProtocol(rc.getProtocol());
      cc.setAddress(rc.getAddress());
      cc.setUsername(rc.getUsername());
      cc.setPassword(rc.getPassword());
      cc.setHighestPriority(false);
      setConfigCenter(cc);
      // 启动 config center
      startConfigCenter();
      return null;
    });
  });
}
void startConfigCenter() {
  if (configCenter == null) {
    ConfigManager.getInstance().getConfigCenter().ifPresent(cc -> this.configCenter = cc);
  }
  if (this.configCenter != null) {
    // 刷新 config center
    this.configCenter.refresh();
    // 准备环境
    prepareEnvironment();
  }
  ConfigManager.getInstance().refreshAll();
}
private void prepareEnvironment() {
  if (configCenter.isValid()) {
    if (!configCenter.checkOrUpdateInited()) {
      return;
    }
    // 获取动态配置信息
    DynamicConfiguration dynamicConfiguration = getDynamicConfiguration(configCenter.toUrl());
    // 获取配置内容
    String configContent = dynamicConfiguration.getProperties(configCenter.getConfigFile(), configCenter.getGroup());
		// 获取 group name
    String appGroup = application != null ? application.getName() : null;
    String appConfigContent = null;
    if (StringUtils.isNotEmpty(appGroup)) {
      appConfigContent = dynamicConfiguration.getProperties
        (StringUtils.isNotEmpty(configCenter.getAppConfigFile()) ? configCenter.getAppConfigFile() : configCenter.getConfigFile(),
         appGroup
        );
    }
    try {
      // 将 config 相关的信息放到 Environment 上
      Environment.getInstance().setConfigCenterFirst(configCenter.isHighestPriority());
      Environment.getInstance().updateExternalConfigurationMap(parseProperties(configContent));
      Environment.getInstance().updateAppExternalConfigurationMap(parseProperties(appConfigContent));
    } catch (IOException e) {
      throw new IllegalStateException("Failed to parse configurations from Config Center.", e);
    }
  }
}
private DynamicConfiguration getDynamicConfiguration(URL url) {
  DynamicConfigurationFactory factory = ExtensionLoader
    .getExtensionLoader(DynamicConfigurationFactory.class)
    .getExtension(url.getProtocol());
  // 使用工厂根据 URL 获取 DynamicConfiguration
  DynamicConfiguration configuration = factory.getDynamicConfiguration(url);
  // 将获取到的 configuration 放到 环境中
  Environment.getInstance().setDynamicConfiguration(configuration);
  return configuration;
}
```

```java
// AbstractDynamicConfigurationFactory
public DynamicConfiguration getDynamicConfiguration(URL url) {
  // 如果 dynamicConfiguration 为空则创建一个
  if (dynamicConfiguration == null) {
    synchronized (this) {
      if (dynamicConfiguration == null) {
        dynamicConfiguration = createDynamicConfiguration(url);
      }
    }
  }
  return dynamicConfiguration;
}
```

```java
// ZookeeperDynamicConfigurationFactory
protected DynamicConfiguration createDynamicConfiguration(URL url) {
  // 如果是 zookeeper 类型则创建一个 ZookeeperDynamicConfiguration 
  return new ZookeeperDynamicConfiguration(url, zookeeperTransporter);
}
```

```java
// ZookeeperDynamicConfiguration
ZookeeperDynamicConfiguration(URL url, ZookeeperTransporter zookeeperTransporter) {
  // 初始化当前类的信息
  this.url = url;
  rootPath = PATH_SEPARATOR + url.getParameter(CONFIG_NAMESPACE_KEY, DEFAULT_GROUP) + "/config";

  initializedLatch = new CountDownLatch(1);
  this.cacheListener = new CacheListener(rootPath, initializedLatch);
  this.executor = Executors.newFixedThreadPool(1, new NamedThreadFactory(this.getClass().getSimpleName(), true));

  // 使用 zookeeperTransporter 连接并得到 zkClient
  zkClient = zookeeperTransporter.connect(url);
  // 监听 /dubbo/config 节点的变化
  zkClient.addDataListener(rootPath, cacheListener, executor);
  try {
    // Wait for connection
    long timeout = url.getParameter("init.timeout", 5000);
    boolean isCountDown = this.initializedLatch.await(timeout, TimeUnit.MILLISECONDS);
    if (!isCountDown) {
      throw new IllegalStateException("Failed to receive INITIALIZED event from zookeeper, pls. check if url "
                                      + url + " is correct");
    }
  } catch (InterruptedException e) {
    logger.warn("Failed to build local cache for config center (zookeeper)." + url);
  }
}
```

```java
// AbstractZookeeperTransporter
public ZookeeperClient connect(URL url) {
  ZookeeperClient zookeeperClient;
  // 获取到所用的连接 URL
  List<String> addressList = getURLBackupAddress(url);
  // The field define the zookeeper server , including protocol, host, port, username, password
  if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null && zookeeperClient.isConnected()) {
    logger.info("find valid zookeeper client from the cache for address: " + url);
    return zookeeperClient;
  }
  // avoid creating too many connections， so add lock
  synchronized (zookeeperClientMap) {
    // 先从缓存中获取
    if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null && zookeeperClient.isConnected()) {
      logger.info("find valid zookeeper client from the cache for address: " + url);
      return zookeeperClient;
    }

    // 从缓存中获取不到则创建一个
    zookeeperClient = createZookeeperClient(url);
    logger.info("No valid zookeeper client found from cache, therefore create a new client for url. " + url);
    // 把连接放到 zookeeperClientMap 中
    writeToClientMap(addressList, zookeeperClient);
  }
  return zookeeperClient;
}
```

```java
// CuratorZookeeperTransporter
public ZookeeperClient createZookeeperClient(URL url) {
  // 创建一个 CuratorZookeeperClient
  return new CuratorZookeeperClient(url);
}
```

```java
// CuratorZookeeperClient
public CuratorZookeeperClient(URL url) {
  super(url);
  try {
    // 从 url 中获取连接需要的信息
    int timeout = url.getParameter(TIMEOUT_KEY, DEFAULT_CONNECTION_TIMEOUT_MS);
    int sessionExpireMs = url.getParameter(ZK_SESSION_EXPIRE_KEY, DEFAULT_SESSION_TIMEOUT_MS);
    // 创建设置相关的属性，超时时间，重处理策略
    CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
      .connectString(url.getBackupAddress())
      .retryPolicy(new RetryNTimes(1, 1000))
      .connectionTimeoutMs(timeout)
      .sessionTimeoutMs(sessionExpireMs);
    String authority = url.getAuthority();
    if (authority != null && authority.length() > 0) {
      builder = builder.authorization("digest", authority.getBytes());
    }
    // 使用 builder 模式创建 Client
    client = builder.build();
    // 在 Client 上添加 listener
    client.getConnectionStateListenable().addListener(new CuratorConnectionStateListener(url));
    // 开始连接
    client.start();
    boolean connected = client.blockUntilConnected(timeout, TimeUnit.MILLISECONDS);
    if (!connected) {
      throw new IllegalStateException("zookeeper not connected");
    }
  } catch (Exception e) {
    throw new IllegalStateException(e.getMessage(), e);
  }
}
```

```java
// RegistryProtocol
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
  // 获取 注册中心的 url
  URL registryUrl = getRegistryUrl(originInvoker);
  // 获取 provider 的 url
  URL providerUrl = getProviderUrl(originInvoker);

  // Subscribe the override data
  // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
  //  the same service. Because the subscribed is cached key with the name of the service, it causes the
  //  subscription information to cover.
  final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
  final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
  overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
	// 使用配置覆盖 URL
  providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
  // 暴露 invoker 
  final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

  // 创建 zookeeper Registry
  final Registry registry = getRegistry(originInvoker);
  // 获取注册的 Provider 的 URL
  final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
  ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                                                                                               registryUrl, registeredProviderUrl);
  //to judge if we need to delay publish
  boolean register = providerUrl.getParameter(REGISTER_KEY, true);
  if (register) {
    // 注册 provider
    register(registryUrl, registeredProviderUrl);
    providerInvokerWrapper.setReg(true);
  }

  // Deprecated! Subscribe to override rules in 2.6.x or before.
  registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

  exporter.setRegisterUrl(registeredProviderUrl);
  exporter.setSubscribeUrl(overrideSubscribeUrl);
  //Ensure that a new exporter instance is returned every time export
  return new DestroyableExporter<>(exporter);
}
public void register(URL registryUrl, URL registeredProviderUrl) {
  // 获取注册中心
  Registry registry = registryFactory.getRegistry(registryUrl);
  // 注册 Provider 的 URL
  registry.register(registeredProviderUrl);
}
```

```java
// DubboProtocol
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
  URL url = invoker.getUrl();

  // export service.
  String key = serviceKey(url);
  DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
  exporterMap.put(key, exporter);

  //export an stub service for dispatching event
  Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
  Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
  if (isStubSupportEvent && !isCallbackservice) {
    String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
    if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
      if (logger.isWarnEnabled()) {
        logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                                              "], has set stubproxy support event ,but no stub methods founded."));
      }

    } else {
      stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
    }
  }

  openServer(url);
  optimizeSerialization(url);

  return exporter;
}
private void openServer(URL url) {
  // find server.
  String key = url.getAddress();
  //client can export a service which's only for server to invoke
  boolean isServer = url.getParameter(IS_SERVER_KEY, true);
  if (isServer) {
    // 获取 Server 
    ExchangeServer server = serverMap.get(key);
    if (server == null) {
      synchronized (this) {
        server = serverMap.get(key);
        if (server == null) {
          // 创建 Server
          serverMap.put(key, createServer(url));
        }
      }
    } else {
      // server supports reset, use together with override
      server.reset(url);
    }
  }
}
private ExchangeServer createServer(URL url) {
  // Build URL
  url = URLBuilder.from(url)
    // send readonly event when server closes, it's enabled by default
    .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
    // enable heartbeat by default
    .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
    .addParameter(CODEC_KEY, DubboCodec.NAME)
    .build();
  String str = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);

  if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
    throw new RpcException("Unsupported server type: " + str + ", url: " + url);
  }

  ExchangeServer server;
  try {
    // 绑定 url 和 处理 Handler
    server = Exchangers.bind(url, requestHandler);
  } catch (RemotingException e) {
    throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
  }

  str = url.getParameter(CLIENT_KEY);
  if (str != null && str.length() > 0) {
    Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
    if (!supportedTypes.contains(str)) {
      throw new RpcException("Unsupported client type: " + str);
    }
  }

  return server;
}
```

```java
// Exchangers
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
  if (url == null) {
    throw new IllegalArgumentException("url == null");
  }
  if (handler == null) {
    throw new IllegalArgumentException("handler == null");
  }
  url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
  // 绑定
  return getExchanger(url).bind(url, handler);
}
```

```java
// HeaderExchanger
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
  // 创建一个 HeaderExchangeServer
  return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```

```java
// Transporters
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
  if (url == null) {
    throw new IllegalArgumentException("url == null");
  }
  if (handlers == null || handlers.length == 0) {
    throw new IllegalArgumentException("handlers == null");
  }
  ChannelHandler handler;
  if (handlers.length == 1) {
    handler = handlers[0];
  } else {
    handler = new ChannelHandlerDispatcher(handlers);
  }
  // 使用 Transporter 绑定 Server
  return getTransporter().bind(url, handler);
}
```

```java
// NettyTransporter
public Server bind(URL url, ChannelHandler listener) throws RemotingException {
  // 底层使用 Netty 创建 Server
  return new NettyServer(url, listener);
}
```

```java
// ProviderConsumerRegTable
public static <T> ProviderInvokerWrapper<T> registerProvider(Invoker<T> invoker, URL registryUrl, URL providerUrl) {
  // 包装 invoker
  ProviderInvokerWrapper<T> wrapperInvoker = new ProviderInvokerWrapper<>(invoker, registryUrl, providerUrl);
  String serviceUniqueName = providerUrl.getServiceKey();
  ConcurrentMap<Invoker, ProviderInvokerWrapper> invokers = providerInvokers.get(serviceUniqueName);
  if (invokers == null) {
    // 放到 providerInvokers 中
    providerInvokers.putIfAbsent(serviceUniqueName, new ConcurrentHashMap<>());
    invokers = providerInvokers.get(serviceUniqueName);
  }
  invokers.put(invoker, wrapperInvoker);
  return wrapperInvoker;
}
```

```java

```

```java
// FailbackRegistry
public void register(URL url) {
  // 放到已经注册的列表中
  super.register(url);
  removeFailedRegistered(url);
  removeFailedUnregistered(url);
  try {
    // Sending a registration request to the server side
    doRegister(url);
  } catch (Exception e) {
    Throwable t = e;

    // If the startup detection is opened, the Exception is thrown directly.
    boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
      && url.getParameter(Constants.CHECK_KEY, true)
      && !CONSUMER_PROTOCOL.equals(url.getProtocol());
    boolean skipFailback = t instanceof SkipFailbackWrapperException;
    if (check || skipFailback) {
      if (skipFailback) {
        t = t.getCause();
      }
      throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
    } else {
      logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
    }

    // Record a failed registration request to a failed list, retry regularly
    addFailedRegistered(url);
  }
}
```

```java
// ZookeeperRegistry
public void doRegister(URL url) {
  try {
    // 注册到 zookeeper 中
    zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
  } catch (Throwable e) {
    throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
  }
}
```

此外是在 `ServiceAnnotationBeanPostProcessor` 中将所有 使用 Dubbo 注解 Service 且在被扫描包里的 Service 使用 `ServiceBean` 为基类注册到容器中。

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
  Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
  if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
    // 扫描包
    registerServiceBeans(resolvedPackagesToScan, registry);
  } else {
    if (logger.isWarnEnabled()) {
      logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
    }
  }
}

private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
  DubboClassPathBeanDefinitionScanner scanner =
    new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
  // 获取 Bean Name Generator
  BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
  scanner.setBeanNameGenerator(beanNameGenerator);
  scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
  scanner.addIncludeFilter(new AnnotationTypeFilter(com.alibaba.dubbo.config.annotation.Service.class));
  for (String packageToScan : packagesToScan) {
    // Registers @Service Bean first
    scanner.scan(packageToScan);
    // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
    // 找到 Bean 定义的 Holder
    Set<BeanDefinitionHolder> beanDefinitionHolders =
      findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);
    if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
      for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
        // 注册到容器中
        registerServiceBean(beanDefinitionHolder, registry, scanner);
      }
      if (logger.isInfoEnabled()) {
        logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                    beanDefinitionHolders +
                    " } were scanned under package[" + packageToScan + "]");
      }
    } else {
      if (logger.isWarnEnabled()) {
        logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                    + packageToScan + "]");
      }
    }
  }
}
```



### 服务引用

对于 Reference 则封装成 `ReferenceBean` Bean。

在创建 Service 类的时候如果有使用 Reference 注解注释的时候 AutoWired 组件，会使用 `AnnotationInjectedBeanPostProcessor` 去创建或者获取需要的组件。

```java
// AnnotationInjectedBeanPostProcessor
// 为 Bean inject 属性，Bean 是 Target 的 Bean，Bean Name 是属性名称，pvs 是属性值
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
  Class<?> injectedType = field.getType();
  // 获取 injected Object
  Object injectedObject = getInjectedObject(attributes, bean, beanName, injectedType, this);
  ReflectionUtils.makeAccessible(field);
  field.set(bean, injectedObject);
}
protected Object getInjectedObject(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
  // 获取 Cache Key
  String cacheKey = buildInjectedObjectCacheKey(attributes, bean, beanName, injectedType, injectedElement);
  // 尝试从 Cache 中获取
  Object injectedObject = injectedObjectsCache.get(cacheKey);
  if (injectedObject == null) {
    // 执行获取 Inject Bean 
    injectedObject = doGetInjectedBean(attributes, bean, beanName, injectedType, injectedElement);
    // Customized inject-object if necessary
    injectedObjectsCache.putIfAbsent(cacheKey, injectedObject);
  }
  return injectedObject;
}
```

```java
// ReferenceAnnotationBeanPostProcessor
protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
  // The name of bean that annotated Dubbo's {@link Service @Service} in local Spring {@link ApplicationContext}
  // Referenced Bean 的名称：ServiceBean:com.xinyue.dubbo.api.service.ArticleService
  String referencedBeanName = buildReferencedBeanName(attributes, injectedType);
  // The name of bean that is declared by {@link Reference @Reference} annotation injection
  // Reference Bean 名称：@Reference com.xinyue.dubbo.api.service.ArticleService
  String referenceBeanName = getReferenceBeanName(attributes, injectedType);
  // Build Reference Bean
  ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referenceBeanName, attributes, injectedType);
  // Register an instance of ReferenceBean as a Spring Bean
  registerReferenceBean(referencedBeanName, referenceBean, attributes, injectedType);
  // 放到缓存中
  cacheInjectedReferenceBean(referenceBean, injectedElement);
  // Get or Create a proxy of ReferenceBean for the specified the type of Dubbo service interface
  return getOrCreateProxy(referencedBeanName, referenceBeanName, referenceBean, injectedType);
}
private void registerReferenceBean(String referencedBeanName, ReferenceBean referenceBean,
                                   AnnotationAttributes attributes,
                                   Class<?> interfaceClass) {
  // 获取 Bean Factory
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  // 获取 Bean 名称
  String beanName = getReferenceBeanName(attributes, interfaceClass);
  if (existsServiceBean(referencedBeanName)) { 
    // If @Service bean is local one
    // Get  the @Service's BeanDefinition from {@link BeanFactory}
    AbstractBeanDefinition beanDefinition = (AbstractBeanDefinition) beanFactory.getBeanDefinition(referencedBeanName);
    RuntimeBeanReference runtimeBeanReference = (RuntimeBeanReference) beanDefinition.getPropertyValues().get("ref");
    // The name of bean annotated @Service
    String serviceBeanName = runtimeBeanReference.getBeanName();
    // register Alias rather than a new bean name, in order to reduce duplicated beans
    beanFactory.registerAlias(serviceBeanName, beanName);
  } else { 
    // Remote @Service Bean
    if (!beanFactory.containsBean(beanName)) {
      // 注册到 Bean 工厂中
      beanFactory.registerSingleton(beanName, referenceBean);
    }
  }
}
private Object getOrCreateProxy(String referencedBeanName, String referenceBeanName, ReferenceBean referenceBean, Class<?> serviceInterfaceType) {
  if (existsServiceBean(referencedBeanName)) { 
    // If the local @Service Bean exists, build a proxy of ReferenceBean
    return newProxyInstance(getClassLoader(), new Class[]{serviceInterfaceType},
                            wrapInvocationHandler(referenceBeanName, referenceBean));
  } else {    
    // ReferenceBean should be initialized and get immediately
    return referenceBean.get();
  }
}
```

```java
// ReferenceConfig
public synchronized T get() {
  // Check each config modules are created properly and override their properties if necessary.
  // 和 ServiceConfig 中的类似
  checkAndUpdateSubConfigs();
  if (destroyed) {
    throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
  }
  if (ref == null) {
    init();
  }
  return ref;
}
public void checkAndUpdateSubConfigs() {
  if (StringUtils.isEmpty(interfaceName)) {
    throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
  }
  // Complete Config
  completeCompoundConfigs();
  // 启动 config center, 刷新 Config
  startConfigCenter();
  // get consumer's global configuration
  checkDefault();
  // 刷新 Reference Config
  this.refresh();
  if (getGeneric() == null && getConsumer() != null) {
    setGeneric(getConsumer().getGeneric());
  }
  // 一些校验
  if (ProtocolUtils.isGeneric(getGeneric())) {
    interfaceClass = GenericService.class;
  } else {
    try {
      interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                                     .getContextClassLoader());
    } catch (ClassNotFoundException e) {
      throw new IllegalStateException(e.getMessage(), e);
    }
    checkInterfaceAndMethods(interfaceClass, methods);
  }
  resolveFile();
  checkApplication();
  checkMetadataReport();
}

private void init() {
  if (initialized) {
    return;
  }
  // Legitimacy check of stub, note that: the local will deprecated, and replace with stub
  checkStubAndLocal(interfaceClass);
  // Check Mock
  checkMock(interfaceClass);
  // Build 参数
  Map<String, String> map = new HashMap<String, String>();
  map.put(SIDE_KEY, CONSUMER_SIDE);
  appendRuntimeParameters(map);
  if (!ProtocolUtils.isGeneric(getGeneric())) {
    String revision = Version.getVersion(interfaceClass, version);
    if (revision != null && revision.length() > 0) {
      map.put(REVISION_KEY, revision);
    }

    String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
    if (methods.length == 0) {
      logger.warn("No method found in service interface " + interfaceClass.getName());
      map.put(METHODS_KEY, ANY_VALUE);
    } else {
      map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), COMMA_SEPARATOR));
    }
  }
  map.put(INTERFACE_KEY, interfaceName);
  appendParameters(map, metrics);
  appendParameters(map, application);
  appendParameters(map, module);
  // remove 'default.' prefix for configs from ConsumerConfig
  // appendParameters(map, consumer, Constants.DEFAULT_KEY);
  appendParameters(map, consumer);
  appendParameters(map, this);
  Map<String, Object> attributes = null;
  if (CollectionUtils.isNotEmpty(methods)) {
    attributes = new HashMap<String, Object>();
    for (MethodConfig methodConfig : methods) {
      appendParameters(map, methodConfig, methodConfig.getName());
      String retryKey = methodConfig.getName() + ".retry";
      if (map.containsKey(retryKey)) {
        String retryValue = map.remove(retryKey);
        if ("false".equals(retryValue)) {
          map.put(methodConfig.getName() + ".retries", "0");
        }
      }
      attributes.put(methodConfig.getName(), convertMethodConfig2AsyncInfo(methodConfig));
    }
  }

  String hostToRegistry = ConfigUtils.getSystemProperty(DUBBO_IP_TO_REGISTRY);
  if (StringUtils.isEmpty(hostToRegistry)) {
    hostToRegistry = NetUtils.getLocalHost();
  } else if (isInvalidLocalHost(hostToRegistry)) {
    throw new IllegalArgumentException("Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
  }
  map.put(REGISTER_IP_KEY, hostToRegistry);

  // 创建代理
  ref = createProxy(map);

  // 生成 key 并 init Customer Model
  String serviceKey = URL.buildKey(interfaceName, group, version);
  ApplicationModel.initConsumerModel(serviceKey, buildConsumerModel(serviceKey, attributes));
  initialized = true;
}
private T createProxy(Map<String, String> map) {
  if (shouldJvmRefer(map)) {
    // Ref JVM
    URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
    invoker = REF_PROTOCOL.refer(interfaceClass, url);
    if (logger.isInfoEnabled()) {
      logger.info("Using injvm service " + interfaceClass.getName());
    }
  } else {
    urls.clear(); // reference retry init will add url to urls, lead to OOM
    if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
      String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
      if (us != null && us.length > 0) {
        for (String u : us) {
          URL url = URL.valueOf(u);
          if (StringUtils.isEmpty(url.getPath())) {
            url = url.setPath(interfaceName);
          }
          if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
          } else {
            urls.add(ClusterUtils.mergeUrl(url, map));
          }
        }
      }
    } else { // assemble URL from register center's configuration
      // if protocols not injvm checkRegistry
      if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())){
        // 和 Service Config 类似也是找到注册中心配置，转换校验，然后连接获取 动态的配置
        checkRegistry();
        // 获取到所有注册中心 URL
        List<URL> us = loadRegistries(false);
        if (CollectionUtils.isNotEmpty(us)) {
          for (URL u : us) {
            URL monitorUrl = loadMonitor(u);
            if (monitorUrl != null) {
              map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
            }
            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
          }
        }
        if (urls.isEmpty()) {
          throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
        }
      }
    }

    if (urls.size() == 1) {
      // 如果只有一个注册中心的 URL 则直接获取即可 invoker
      // 会调用到 RegistryProtocol 的 refer
      invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
    } else {
      // 如果有多个则遍历获取
      List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
      URL registryURL = null;
      for (URL url : urls) {
        invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
        if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
          registryURL = url; // use last registry url
        }
      }
      // 获取一个 invoker 集合
      if (registryURL != null) { // registry url is available
        // use RegistryAwareCluster only when register's CLUSTER is available
        URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
        // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
        invoker = CLUSTER.join(new StaticDirectory(u, invokers));
      } else { // not a registry url, must be direct invoke.
        invoker = CLUSTER.join(new StaticDirectory(invokers));
      }
    }
  }

  if (shouldCheck() && !invoker.isAvailable()) {
    throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
  }
  if (logger.isInfoEnabled()) {
    logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
  }
  
  // 元信息发布
  MetadataReportService metadataReportService = null;
  if ((metadataReportService = getMetadataReportService()) != null) {
    URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
    metadataReportService.publishConsumer(consumerURL);
  }
  // create service proxy
  return (T) PROXY_FACTORY.getProxy(invoker);
}
protected List<URL> loadRegistries(boolean provider) {
  // check && override if necessary
  List<URL> registryList = new ArrayList<URL>();
  // 遍历注册中心
  if (CollectionUtils.isNotEmpty(registries)) {
    for (RegistryConfig config : registries) {
      String address = config.getAddress();
      if (StringUtils.isEmpty(address)) {
        address = ANYHOST_VALUE;
      }
      if (!RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
        Map<String, String> map = new HashMap<String, String>();
        appendParameters(map, application);
        appendParameters(map, config);
        map.put(PATH_KEY, RegistryService.class.getName());
        appendRuntimeParameters(map);
        if (!map.containsKey(PROTOCOL_KEY)) {
          map.put(PROTOCOL_KEY, DUBBO_PROTOCOL);
        }
        List<URL> urls = UrlUtils.parseURLs(address, map);

        // 找到所有的注册中心 URL 并返回
        for (URL url : urls) {
          url = URLBuilder.from(url)
            .addParameter(REGISTRY_KEY, url.getProtocol())
            .setProtocol(REGISTRY_PROTOCOL)
            .build();
          if ((provider && url.getParameter(REGISTER_KEY, true))
              || (!provider && url.getParameter(SUBSCRIBE_KEY, true))) {
            registryList.add(url);
          }
        }
      }
    }
  }
  return registryList;
}
```

```java
// RegistryProtocol
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
  // 生成 URL
  url = URLBuilder.from(url)
    .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
    .removeParameter(REGISTRY_KEY)
    .build();
  // 根据 URL 获取注册中心
  Registry registry = registryFactory.getRegistry(url);
  if (RegistryService.class.equals(type)) {
    // 如果是注册中心 Service 则通过 ProxyFactory 创建 invoker
    return proxyFactory.getInvoker((T) registry, type, url);
  }

  // group="a,b" or group="*"
  Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
  String group = qs.get(GROUP_KEY);
  if (group != null && group.length() > 0) {
    if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
      return doRefer(getMergeableCluster(), registry, type, url);
    }
  }
  // 执行获取
  return doRefer(cluster, registry, type, url);
}
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
  // 新建一个注册发现根据需要 provider 的 type 和 当前的 url
  RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
  // 设置注册中心和协议
  directory.setRegistry(registry);
  directory.setProtocol(protocol);
  // all attributes of REFER_KEY
  Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
  // 生成订阅者的 URL
  URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
  if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
    directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
    // 注册 Customer
    registry.register(directory.getRegisteredConsumerUrl());
  }
  directory.buildRouterChain(subscribeUrl);
  // 开始订阅
  directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                                                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));

  // 订阅后就拿到了 invoker
  Invoker invoker = cluster.join(directory);
  // 注册到 ProviderConsumerRegTable 中
  ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
  return invoker;
}
```

```java
// RegistryDirectory
public void subscribe(URL url) {
  // 设置 Consumer 的 URL
  setConsumerUrl(url);
  CONSUMER_CONFIGURATION_LISTENER.addNotifyListener(this);
  serviceConfigurationListener = new ReferenceConfigurationListener(this, url);
  registry.subscribe(url, this);
}
public synchronized void notify(List<URL> urls) {
  // 获取到的 provider 的url
  Map<String, List<URL>> categoryUrls = urls.stream()
    .filter(Objects::nonNull)
    .filter(this::isValidCategory)
    .filter(this::isNotCompatibleFor26x)
    .collect(Collectors.groupingBy(url -> {
      if (UrlUtils.isConfigurator(url)) {
        return CONFIGURATORS_CATEGORY;
      } else if (UrlUtils.isRoute(url)) {
        return ROUTERS_CATEGORY;
      } else if (UrlUtils.isProvider(url)) {
        return PROVIDERS_CATEGORY;
      }
      return "";
    }));

  List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
  this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

  List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
  toRouters(routerURLs).ifPresent(this::addRouters);

  // providers
  List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
  // 刷新和覆盖 invoker
  refreshOverrideAndInvoker(providerURLs);
}
private void refreshOverrideAndInvoker(List<URL> urls) {
  // mock zookeeper://xxx?mock=return null
  overrideDirectoryUrl();
  // 根据 URL 刷卡 invoker
  refreshInvoker(urls);
}
private void refreshInvoker(List<URL> invokerUrls) {
  Assert.notNull(invokerUrls, "invokerUrls should not be null");

  if (invokerUrls.size() == 1
      && invokerUrls.get(0) != null
      && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
    this.forbidden = true; // Forbid to access
    this.invokers = Collections.emptyList();
    routerChain.setInvokers(this.invokers);
    destroyAllInvokers(); // Close all invokers
  } else {
    this.forbidden = false; // Allow to access
    Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
    if (invokerUrls == Collections.<URL>emptyList()) {
      invokerUrls = new ArrayList<>();
    }
    if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
      invokerUrls.addAll(this.cachedInvokerUrls);
    } else {
      this.cachedInvokerUrls = new HashSet<>();
      this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
    }
    if (invokerUrls.isEmpty()) {
      return;
    }
    // 根据 URL 获取 Invoker
    Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

    /**
             * If the calculation is wrong, it is not processed.
             *
             * 1. The protocol configured by the client is inconsistent with the protocol of the server.
             *    eg: consumer protocol = dubbo, provider only has other protocol services(rest).
             * 2. The registration center is not robust and pushes illegal specification data.
             *
             */
    if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
      logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                                             .toString()));
      return;
    }

    List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
    // pre-route and build cache, notice that route cache should build on original Invoker list.
    // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
    routerChain.setInvokers(newInvokers);
    this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
    this.urlInvokerMap = newUrlInvokerMap;

    try {
      destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
    } catch (Exception e) {
      logger.warn("destroyUnusedInvokers error. ", e);
    }
  }
}
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
  Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();
  if (urls == null || urls.isEmpty()) {
    return newUrlInvokerMap;
  }
  Set<String> keys = new HashSet<>();
  String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
  for (URL providerUrl : urls) {
    // If protocol is configured at the reference side, only the matching protocol is selected
    if (queryProtocols != null && queryProtocols.length() > 0) {
      boolean accept = false;
      String[] acceptProtocols = queryProtocols.split(",");
      for (String acceptProtocol : acceptProtocols) {
        if (providerUrl.getProtocol().equals(acceptProtocol)) {
          accept = true;
          break;
        }
      }
      if (!accept) {
        continue;
      }
    }
    if (EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
      continue;
    }
    if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
      logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() +
                                             " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() +
                                             " to consumer " + NetUtils.getLocalHost() + ", supported protocol: " +
                                             ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
      continue;
    }
    URL url = mergeUrl(providerUrl);

    String key = url.toFullString(); // The parameter urls are sorted
    if (keys.contains(key)) { // Repeated url
      continue;
    }
    keys.add(key);
    // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
    Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
    Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
    if (invoker == null) { // Not in the cache, refer again
      try {
        boolean enabled = true;
        if (url.hasParameter(DISABLED_KEY)) {
          enabled = !url.getParameter(DISABLED_KEY, false);
        } else {
          enabled = url.getParameter(ENABLED_KEY, true);
        }
        if (enabled) {
          // 创建 invoker delegate，最终创建的是一个 dubbo 的 config
          invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);
        }
      } catch (Throwable t) {
        logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
      }
      if (invoker != null) { // Put new invoker in cache
        newUrlInvokerMap.put(key, invoker);
      }
    } else {
      newUrlInvokerMap.put(key, invoker);
    }
  }
  keys.clear();
  return newUrlInvokerMap;
}
```

```java
// FailbackRegistry
public void subscribe(URL url, NotifyListener listener) {
  super.subscribe(url, listener);
  removeFailedSubscribed(url, listener);
  try {
    // Sending a subscription request to the server side
    doSubscribe(url, listener);
  } catch (Exception e) {
    Throwable t = e;

    List<URL> urls = getCacheUrls(url);
    if (CollectionUtils.isNotEmpty(urls)) {
      notify(url, listener, urls);
      logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
    } else {
      // If the startup detection is opened, the Exception is thrown directly.
      boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
        && url.getParameter(Constants.CHECK_KEY, true);
      boolean skipFailback = t instanceof SkipFailbackWrapperException;
      if (check || skipFailback) {
        if (skipFailback) {
          t = t.getCause();
        }
        throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
      } else {
        logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
      }
    }

    // Record a failed registration request to a failed list, retry regularly
    addFailedSubscribed(url, listener);
  }
}
// 去订阅
public void doSubscribe(final URL url, final NotifyListener listener) {
  try {
    if (ANY_VALUE.equals(url.getServiceInterface())) {
      String root = toRootPath();
      ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
      if (listeners == null) {
        zkListeners.putIfAbsent(url, new ConcurrentHashMap<>());
        listeners = zkListeners.get(url);
      }
      ChildListener zkListener = listeners.get(listener);
      if (zkListener == null) {
        listeners.putIfAbsent(listener, (parentPath, currentChilds) -> {
          for (String child : currentChilds) {
            child = URL.decode(child);
            if (!anyServices.contains(child)) {
              anyServices.add(child);
              subscribe(url.setPath(child).addParameters(INTERFACE_KEY, child,
                                                         Constants.CHECK_KEY, String.valueOf(false)), listener);
            }
          }
        });
        zkListener = listeners.get(listener);
      }
      zkClient.create(root, false);
      List<String> services = zkClient.addChildListener(root, zkListener);
      if (CollectionUtils.isNotEmpty(services)) {
        for (String service : services) {
          service = URL.decode(service);
          anyServices.add(service);
          subscribe(url.setPath(service).addParameters(INTERFACE_KEY, service,
                                                       Constants.CHECK_KEY, String.valueOf(false)), listener);
        }
      }
    } else {
      // 如果有订阅的接口
      List<URL> urls = new ArrayList<>();
      for (String path : toCategoriesPath(url)) {
        // 获取 listener
        ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
        if (listeners == null) {
          zkListeners.putIfAbsent(url, new ConcurrentHashMap<>());
          // 从注册中心中获取 listener
          listeners = zkListeners.get(url);
        }
        ChildListener zkListener = listeners.get(listener);
        if (zkListener == null) {
          listeners.putIfAbsent(listener, (parentPath, currentChilds) -> ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds)));
          // 获取 listener
          zkListener = listeners.get(listener);
        }
        // 在注册中心中创建订阅
        zkClient.create(path, false);
        List<String> children = zkClient.addChildListener(path, zkListener);
        if (children != null) {
          urls.addAll(toUrlsWithEmpty(url, path, children));
        }
      }
      // 通过获取到了 listener
      notify(url, listener, urls);
    }
  } catch (Throwable e) {
    throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
  }
}
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
  if (url == null) {
    throw new IllegalArgumentException("notify url == null");
  }
  if (listener == null) {
    throw new IllegalArgumentException("notify listener == null");
  }
  try {
    doNotify(url, listener, urls);
  } catch (Exception t) {
    // Record a failed registration request to a failed list, retry regularly
    addFailedNotified(url, listener, urls);
    logger.error("Failed to notify for subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
  }
}
protected void doNotify(URL url, NotifyListener listener, List<URL> urls) {
  super.notify(url, listener, urls);
}
```

```java
// AbstractRegistry
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
  if (url == null) {
    throw new IllegalArgumentException("notify url == null");
  }
  if (listener == null) {
    throw new IllegalArgumentException("notify listener == null");
  }
  if ((CollectionUtils.isEmpty(urls))
      && !ANY_VALUE.equals(url.getServiceInterface())) {
    logger.warn("Ignore empty notify urls for subscribe url " + url);
    return;
  }
  if (logger.isInfoEnabled()) {
    logger.info("Notify urls for subscribe url " + url + ", urls: " + urls);
  }
  // keep every provider's category.
  Map<String, List<URL>> result = new HashMap<>();
  for (URL u : urls) {
    if (UrlUtils.isMatch(url, u)) {
      String category = u.getParameter(CATEGORY_KEY, DEFAULT_CATEGORY);
      List<URL> categoryList = result.computeIfAbsent(category, k -> new ArrayList<>());
      categoryList.add(u);
    }
  }
  if (result.size() == 0) {
    return;
  }
  Map<String, List<URL>> categoryNotified = notified.computeIfAbsent(url, u -> new ConcurrentHashMap<>());
  for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
    String category = entry.getKey();
    List<URL> categoryList = entry.getValue();
    categoryNotified.put(category, categoryList);
    // 调用 listener 的 notify 方法
    listener.notify(categoryList);
    // We will update our cache file after each notification.
    // When our Registry has a subscribe failure due to network jitter, we can return at least the existing cache URL.
    saveProperties(url);
  }
}
```

```java
// DubboProtocol
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
  optimizeSerialization(url);

  // create rpc invoker.
  DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
  invokers.add(invoker);

  return invoker;
}
// 获取客户端地址
private ExchangeClient[] getClients(URL url) {
  // whether to share connection
  boolean useShareConnect = false;
  int connections = url.getParameter(CONNECTIONS_KEY, 0);
  List<ReferenceCountExchangeClient> shareClients = null;
  // if not configured, connection is shared, otherwise, one connection for one service
  if (connections == 0) {
    useShareConnect = true;
    // The xml configuration should have a higher priority than properties.
    String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
    connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
    shareClients = getSharedClient(url, connections);
  }
  ExchangeClient[] clients = new ExchangeClient[connections];
  for (int i = 0; i < clients.length; i++) {
    if (useShareConnect) {
      clients[i] = shareClients.get(i);
    } else {
      clients[i] = initClient(url);
    }
  }
  return clients;
}
private ExchangeClient initClient(URL url) {
  // client type setting.
  String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
  url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
  // enable heartbeat by default
  url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));
  // BIO is not allowed since it has severe performance issue.
  if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
    throw new RpcException("Unsupported client type: " + str + "," +
                           " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
  }
  ExchangeClient client;
  try {
    // connection should be lazy
    if (url.getParameter(LAZY_CONNECT_KEY, false)) {
      client = new LazyConnectExchangeClient(url, requestHandler);
    } else {
      // 最终创建了一个 NettyClient 的连接
      client = Exchangers.connect(url, requestHandler);
    }
  } catch (RemotingException e) {
    throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
  }
  return client;
}
```

### 服务调用





```
ServiceBean
DubboProtocol
Protocol
RegistryProtocol
ServieConfig
ReferenceBean
```

