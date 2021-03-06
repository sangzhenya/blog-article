---
title: "Spring Boot 嵌入式容器"
tags: ["Spring", "Spring Boot"]
categories: ["Spring"]
date: "2019-02-23T09:00:00+08:00"
---

### 对于嵌入式容器的配置

Spring 的基本的 Server 的配置都可以在 `application.yaml` 中通过 `server` 相关的属性进行配置。

![Server 相关的配置](http://img.programya.com/Snipaste_2019-12-16_23-11-06.png)

其对应的配置类是 

```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {
  private final Tomcat tomcat = new Tomcat();
  
  public static class Tomcat {
		private final Accesslog accesslog = new Accesslog();
		private String protocolHeader;
		private String protocolHeaderHttpsValue = "https";
		private String portHeader = "X-Forwarded-Port";
  }
}
```

其内部类 `Tomcat` 用户配置 `Tomcat` 相关的信息。在 `ServletWebServerFactoryCustomizer` 会使用 其中的配置对 嵌入式容器进行配置， 同样也可以通过使用 `WebServerFactoryCustomizer` 去配置嵌入式容器，例如如下：

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Bean
    public WebServerFactoryCustomizer<ConfigurableWebServerFactory> customizer() {
        return factory -> factory.setPort(8081);
    }
}
```

### 在嵌入式容器中使用 Web 三大组件

#### Servlet

可以使用 `ServletRegistrationBean` 进行注册。

```java
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.getWriter().write("Hello World");
    }
}

@Bean
public ServletRegistrationBean<MyServlet> myServlet() {
  ServletRegistrationBean<MyServlet> servletRegistrationBean = new ServletRegistrationBean<>(new MyServlet(), "/servlet");
  servletRegistrationBean.setLoadOnStartup(1);
  return servletRegistrationBean;
}
```



#### Filter

可以使用 `FilterRegistrationBean` 进行注册。

```java
public class MyFilter extends HttpFilter {
    private Log log = LogFactory.getLog(MyFilter.class);
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("My Filter Processed...");
        chain.doFilter(request, response);
    }
}

@Bean
public FilterRegistrationBean<MyFilter> myFilter() {
  FilterRegistrationBean<MyFilter> filterFilterRegistrationBean = new FilterRegistrationBean<>();
  filterFilterRegistrationBean.setFilter(new MyFilter());
  filterFilterRegistrationBean.setUrlPatterns(Collections.singletonList("/servlet"));
  return filterFilterRegistrationBean;
}
```



#### Listener

可以使用 `ServletListenerRegistrationBean` 注册。

```java
public class MyListener implements ServletContextListener {
    private Log log = LogFactory.getLog(MyListener.class);
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        log.info("Servlet Context Inited");
    }
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        log.info("Servlet Context Destroyed");
    }
}

@Bean
public ServletListenerRegistrationBean<MyListener> myListener() {
  return new ServletListenerRegistrationBean<>(new MyListener());
}
```

SpringBoot 在自动注册 Spring MVC 的时候会自动注册 `DispatcherServlet`。使用 `DispatcherServletAutoConfiguration` 进行注册。

```java
public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";

@Configuration(proxyBeanMethods = false)
@Conditional(DispatcherServletRegistrationCondition.class)
@ConditionalOnClass(ServletRegistration.class)
@EnableConfigurationProperties(WebMvcProperties.class)
@Import(DispatcherServletConfiguration.class)
protected static class DispatcherServletRegistrationConfiguration {

  @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
  @ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
  public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
                                                                         WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
    DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
                                                                                           webMvcProperties.getServlet().getPath());
    registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
    registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
    multipartConfig.ifAvailable(registration::setMultipartConfig);
    return registration;
  }

}
```



### 切换嵌入式容器

在 Spring Boot 中除了支持 Tomcat 之外也支持 Jetty 和 Undertow 等另外两种 Servlet 容器。其中 Jetty 更适合于长链接的应用，而 Undertow 更适合于高并发的应用。

例如必须想要使用 Undertow 代替默认的 Tomcat 容器，首先在 pom 文件中去除 Tomcat 的依赖，然后加入 Undertow 的依赖即可。

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions>
    <exclusion>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <groupId>org.springframework.boot</groupId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <artifactId>spring-boot-starter-undertow</artifactId>
  <groupId>org.springframework.boot</groupId>
</dependency>
```

### 嵌入式 Servlet 容器自动配置原理

SpringBoot 主要依赖于`ServletWebServerFactoryAutoConfiguration` 引入不同的嵌入式 Servlet。

其会引入各个 Servlet 容器的 Config 类，如下所示：

```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
```

其中 Undertow 而配置如下：

```java
@Configuration(proxyBeanMethods = false)
class ServletWebServerFactoryConfiguration {
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	public static class EmbeddedUndertow {
		// 讲 Undertow 的 Factory （UndertowServletWebServerFactory）添加到容器中，通过其 getWebServer 方法即可获取到对应的 WebServer
		@Bean
		public UndertowServletWebServerFactory undertowServletWebServerFactory(
				ObjectProvider<UndertowDeploymentInfoCustomizer> deploymentInfoCustomizers,
				ObjectProvider<UndertowBuilderCustomizer> builderCustomizers) {
			UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
			factory.getDeploymentInfoCustomizers()
					.addAll(deploymentInfoCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getBuilderCustomizers().addAll(builderCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}

}

```

同时其为容器导入了一个 `ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class` 后置处理器对容器进行处理。

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                    BeanDefinitionRegistry registry) {
  if (this.beanFactory == null) {
    return;
  }
  // 为容器导入了 WebServerFactoryCustomizerBeanPostProcessor 组件
  registerSyntheticBeanIfMissing(registry, "webServerFactoryCustomizerBeanPostProcessor",
                                 WebServerFactoryCustomizerBeanPostProcessor.class);
  registerSyntheticBeanIfMissing(registry, "errorPageRegistrarBeanPostProcessor",
                                 ErrorPageRegistrarBeanPostProcessor.class);
}
```

在 `WebServerFactoryCustomizerBeanPostProcessor` 中会调用 WebServer 容器的自定义配置。

```java
private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
  // 拿到所有的 Customzer 逐个调用 customize 方法对容器进行定制。
  LambdaSafe.callbacks(WebServerFactoryCustomizer.class, getCustomizers(), webServerFactory)
    .withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
    .invoke((customizer) -> customizer.customize(webServerFactory));
}
```

SpringBoot 主要依赖 `EmbeddedWebServerFactoryCustomizerAutoConfiguration` 对嵌入式容器进行自动配置。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication
@EnableConfigurationProperties(ServerProperties.class)
public class EmbeddedWebServerFactoryCustomizerAutoConfiguration {
  // 例如对于 Undertow 的配置
  @Configuration(proxyBeanMethods = false)
  // 在存在 Undertow 和 SslClientAuthMode 才会添加到容器中
	@ConditionalOnClass({ Undertow.class, SslClientAuthMode.class })
	public static class UndertowWebServerFactoryCustomizerConfiguration {
		@Bean
		public UndertowWebServerFactoryCustomizer undertowWebServerFactoryCustomizer(Environment environment,
				ServerProperties serverProperties) {
			return new UndertowWebServerFactoryCustomizer(environment, serverProperties);
		}
	}
}
```

可以看出在 autoConfig 中为容器中加入了一个 WebServerFactoryCustomizer，对 Servlet 容器的配置。SpringBoot 中默认有四个这样的 Customizer，其类结构图如下所示：

![WebServerFactoryCustomizer](http://img.programya.com/Snipaste_2019-12-17_23-22-31.png)

主要是从配置文件中读取相关的配置设置到 WebServer 上。

```java
public class UndertowWebServerFactoryCustomizer
		implements WebServerFactoryCustomizer<ConfigurableUndertowWebServerFactory>, Ordered {

	private final Environment environment;

	private final ServerProperties serverProperties;

	public UndertowWebServerFactoryCustomizer(Environment environment, ServerProperties serverProperties) {
		this.environment = environment;
		this.serverProperties = serverProperties;
	}

	@Override
	public int getOrder() {
		return 0;
	}

	@Override
	public void customize(ConfigurableUndertowWebServerFactory factory) {
    // 从配置文件中配置并设置到容器上
		PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
		FactoryOptions options = new FactoryOptions(factory);
		ServerProperties properties = this.serverProperties;
		map.from(properties::getMaxHttpHeaderSize).asInt(DataSize::toBytes).when(this::isPositive)
				.to(options.server(UndertowOptions.MAX_HEADER_SIZE));
		map.from(properties::getConnectionTimeout).asInt(Duration::toMillis)
				.to(options.server(UndertowOptions.NO_REQUEST_TIMEOUT));
		mapUndertowProperties(factory, options);
		mapAccessLogProperties(factory);
		map.from(this::getOrDeduceUseForwardHeaders).to(factory::setUseForwardHeaders);
	}
}
```



### 嵌入式 Servlet 容器启动原理

首先在主类中调用 `SpringApplication.run` 方法并将当前的类传入作为参数传入，然后经过几次调用会调用 `public ConfigurableApplicationContext run(String... args)` 方法，其中会调用容器的 `refreshContext` 去刷新容器，其中会调用 `refresh` 方法去刷新容器，然后会调用到 `AbstractApplicationContext` 的 `refresh` 方法，也就是 Spring 容器的刷新容器的方法。在其子类 `ServletWebServerApplicationContext` 实现了 `onRefresh` 之后会调用 `createWebServer` 方法去创建 WebServer。

```java
// ServletWebServerApplicationContext
private void createWebServer() {
  // 获取当前的 webServer
  WebServer webServer = this.webServer;
  // 获取当前的 ServletContext
  ServletContext servletContext = getServletContext();
  // 如果均为空则尝去获取 WebServerFactory 然后通过 WebServerFactory 获取 WebServer
  if (webServer == null && servletContext == null) {
    ServletWebServerFactory factory = getWebServerFactory();
    // 在获取到 WebServerFactory 之后调用 getWebServer 去获取 WebServer
    this.webServer = factory.getWebServer(getSelfInitializer());
  }
  else if (servletContext != null) {
    try {
      // 如果 Servlet 不为空则尝试去启动 ServletContext
      getSelfInitializer().onStartup(servletContext);
    }
    catch (ServletException ex) {
      throw new ApplicationContextException("Cannot initialize servlet context", ex);
    }
  }
  initPropertySources();
}

protected ServletWebServerFactory getWebServerFactory() {
  // Use bean names so that we don't consider the hierarchy
  // 获取 WebServerFactory 的 Bean Name
  String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
  if (beanNames.length == 0) {
    throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to missing "
                                          + "ServletWebServerFactory bean.");
  }
  if (beanNames.length > 1) {
    throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to multiple "
                                          + "ServletWebServerFactory beans : " + StringUtils.arrayToCommaDelimitedString(beanNames));
  }
  // 根据 Bean Name 获取继承自 ServletWebServerFactory 的 ServletWebServerFactory
  // 这里也可以看出其实调用 Spring getBean 的流程
  return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}
```

所以其实就是 `getBean`， `doGetBean`，单实例的情况使用 `getSingleton` 方法获取，找不到的情况调用 `createBean` 去创建 Bean 实例，调用 `doCreateBean` 去真正执行创建，其中调用 `createBeanInstance` 去创建 Bean 实例。在其中根据 Bean 的定义判断有相应的创建方法，则可以直接调用相应的方法创建 Bean 实例。

```java
// 方法名称是 undertowServletWebServerFactory
if (mbd.getFactoryMethodName() != null) {
  // 这里的 BeanName 也是 undertowServletWebServerFactory
  return instantiateUsingFactoryMethod(beanName, mbd, args);
}
```

最终会调用到 `ServletWebServerFactoryConfiguration` 中的创建 WebServerFactory 的方法。

```java
@Bean
public UndertowServletWebServerFactory undertowServletWebServerFactory(
  ObjectProvider<UndertowDeploymentInfoCustomizer> deploymentInfoCustomizers,
  ObjectProvider<UndertowBuilderCustomizer> builderCustomizers) {
  UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
  factory.getDeploymentInfoCustomizers()
    .addAll(deploymentInfoCustomizers.orderedStream().collect(Collectors.toList()));
  factory.getBuilderCustomizers().addAll(builderCustomizers.orderedStream().collect(Collectors.toList()));
  return factory;
}
```

在上面创建 Bean 之后会调用 `initializeBean` 去初始化 Bean 对象。在真正初始化之前会逐个调用调用 Bean 后置处理器的 `applyBeanPostProcessorsBeforeInitialization`  方法去执行相应的处理方法，其中就会调用到 `WebServerFactoryCustomizerBeanPostProcessor` 的 `postProcessBeforeInitialization` 方法，对 Servlet 容器进行定制。

```java
private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
  // 获取到所有的 WebServerFactoryCustomizer 逐个调用其 customize 对容器工厂进行定制
  LambdaSafe.callbacks(WebServerFactoryCustomizer.class, getCustomizers(), webServerFactory)
    .withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
    .invoke((customizer) -> customizer.customize(webServerFactory));
}

// 获取到所有的 Customers
private Collection<WebServerFactoryCustomizer<?>> getCustomizers() {
  if (this.customizers == null) {
    // Look up does not include the parent context
    this.customizers = new ArrayList<>(getWebServerFactoryCustomizerBeans());
    this.customizers.sort(AnnotationAwareOrderComparator.INSTANCE);
    this.customizers = Collections.unmodifiableList(this.customizers);
  }
  return this.customizers;
}
```

这里就会调用到 `UndertowWebServerFactoryCustomizer` 的 `customize` 方法了。

在得到 WebServerFactory 之后会调用 `getWebServer` 方法去获取 WebServer。

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
  // 首先去创建一个 DeploymentManager
  DeploymentManager manager = createDeploymentManager(initializers);
  // 获取配置的 端口号
  int port = getPort();
  // 创建 Builder
  Builder builder = createBuilder(port);
  return getUndertowWebServer(builder, manager, port);
}

private DeploymentManager createDeploymentManager(ServletContextInitializer... initializers) {
  // 获取 DeploymentInfo
  DeploymentInfo deployment = Servlets.deployment();
  // 注册 ServletContextInitializer
  registerServletContainerInitializerToDriveServletContextInitializers(deployment, initializers);
  // 设置 ClassLoader
  deployment.setClassLoader(getServletClassLoader());
  // 设置 ContextPath
  deployment.setContextPath(getContextPath());
  deployment.setDisplayName(getDisplayName());
  deployment.setDeploymentName("spring-boot");
  if (isRegisterDefaultServlet()) {
    deployment.addServlet(Servlets.servlet("default", DefaultServlet.class));
  }
  // 继续设置一些其他的信息
  configureErrorPages(deployment);
  deployment.setServletStackTraces(ServletStackTraces.NONE);
  deployment.setResourceManager(getDocumentRootResourceManager());
  deployment.setTempDir(createTempDir("undertow"));
  deployment.setEagerFilterInit(this.eagerInitFilters);
  configureMimeMappings(deployment);
  // 对 deployment 进行定制
  for (UndertowDeploymentInfoCustomizer customizer : this.deploymentInfoCustomizers) {
    customizer.customize(deployment);
  }
  if (isAccessLogEnabled()) {
    configureAccessLog(deployment);
  }
  if (getSession().isPersistent()) {
    File dir = getValidSessionStoreDir();
    deployment.setSessionPersistenceManager(new FileSessionPersistence(dir));
  }
  addLocaleMappings(deployment);
  DeploymentManager manager = Servlets.newContainer().addDeployment(deployment);
  manager.deploy();
  if (manager.getDeployment() instanceof DeploymentImpl) {
    removeSuperfluousMimeMappings((DeploymentImpl) manager.getDeployment(), deployment);
  }
  // 设置 Session 超时时间
  SessionManager sessionManager = manager.getDeployment().getSessionManager();
  Duration timeoutDuration = getSession().getTimeout();
  int sessionTimeout = (isZeroOrLess(timeoutDuration) ? -1 : (int) timeoutDuration.getSeconds());
  sessionManager.setDefaultSessionTimeout(sessionTimeout);
  return manager;
}

private Builder createBuilder(int port) {
  // 获取 Undertow 的 Builder
  Builder builder = Undertow.builder();
  // 设置缓冲区大小
  if (this.bufferSize != null) {
    builder.setBufferSize(this.bufferSize);
  }
  // 设置 IO 线程数
  if (this.ioThreads != null) {
    builder.setIoThreads(this.ioThreads);
  }
  // 设置 work 线程数
  if (this.workerThreads != null) {
    builder.setWorkerThreads(this.workerThreads);
  }
  // 设置直接 buffer
  if (this.directBuffers != null) {
    builder.setDirectBuffers(this.directBuffers);
  }
  // ssl 配置
  if (getSsl() != null && getSsl().isEnabled()) {
    customizeSsl(builder);
  }
  else {
    builder.addHttpListener(port, getListenAddress());
  }
  // 逐个调用 Customizer 去对 Build 定制
  for (UndertowBuilderCustomizer customizer : this.builderCustomizers) {
    customizer.customize(builder);
  }
  return builder;
}

protected UndertowServletWebServer getUndertowWebServer(Builder builder, DeploymentManager manager, int port) {
  // 调用构造方法创建 UndertowServletWebServer 
  return new UndertowServletWebServer(builder, manager, getContextPath(), isUseForwardHeaders(), port >= 0,
                                      getCompression(), getServerHeader());
}
```

```java
// UndertowServletWebServer
public UndertowServletWebServer(Builder builder, DeploymentManager manager, String contextPath,
                                boolean useForwardHeaders, boolean autoStart, Compression compression, String serverHeader) {
  this.builder = builder;
  this.manager = manager;
  this.contextPath = contextPath;
  this.useForwardHeaders = useForwardHeaders;
  this.autoStart = autoStart;
  this.compression = compression;
  this.serverHeader = serverHeader;
}
```
