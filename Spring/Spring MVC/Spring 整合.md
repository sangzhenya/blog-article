## Spring 和 Spring MVC 整合

Web 容器启动的时候会扫描每个 jar 包下的 `META-INF\services\javax.servlet.ServletContainerInitializer` 文件中的内容，根据文件中的定义全类名找到 `ServletContainerInitializer` 的子类，然后调用其 `onStartup` 方法。在引入的 `spring-web`  下就有这样的一个文件，其内容是 `org.springframework.web.SpringServletContainerInitializer`， 即定义了一个  `SpringServletContainerInitializer`， 容器启动的时候会执行该类的 `onStartup` 方法。其内容如下：

```java
// 关注所有 WebApplicationInitializer 的实现类
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<>();

        // 如果存在 WebApplicationInitializer 的实现类
		if (webAppInitializerClasses != null) {
            // 遍历实现类
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
                // 如果不是 接口。抽象的冰倩是 WebApplicationInitializer 类型的
                // 则创建实例加入到 initializers 中
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

        // 如果为空则出错，记录没有 Spring WebApplicationInitializer
		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		// 对 initializers 进行排序
        AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
            // 逐个调用 WebApplicationInitializer 的 onStartup 方法。
			initializer.onStartup(servletContext);
		}
	}
}
```

默认有 4 个抽象类实现了 `WebApplicationInitializer` 接口，如下图所示：

![WebApplicationInitializer](http://img.sangzhenya.com/Snipaste_2019-11-30_19-22-35.png)

最底层的 `AbstractAnnotationConfigDispatcherServletInitializer` 中并没有对 `onStartup` 方法重写，所以看 `AbstractDispatcherServletInitializer` 类，其重写了 父类的 `onStartup` 方法，如下所示：

```java
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
    // 调用父类的 onStartup 方法
    super.onStartup(servletContext);
    // 注册 Dispatcher Servlet
    registerDispatcherServlet(servletContext);
}
```

所以先看以下其他父类 `AbstractContextLoaderInitializer` 中的 `onStartup`  方法。

```java
// AbstractContextLoaderInitializer
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
    // 注册 ContextLoaderListener
    registerContextLoaderListener(servletContext);
}

protected void registerContextLoaderListener(ServletContext servletContext) {
    // 获取 WebApplicationContext，创建根容器
    WebApplicationContext rootAppContext = createRootApplicationContext();
    if (rootAppContext != null) {
        // 如果获取到了则根据获取到的 WebApplicationContext 
        // 新建一个 ContextLoaderListener
        ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
        // 添加其他的 ContextInitializers，默认是空
        listener.setContextInitializers(getRootApplicationContextInitializers());
        // 动态添加到 Servlet Context 中
        servletContext.addListener(listener);
    }
    else {
        logger.debug("No ContextLoaderListener registered, as " +
                     "createRootApplicationContext() did not return an application context");
    }
}

protected abstract WebApplicationContext createRootApplicationContext();

@Nullable
protected ApplicationContextInitializer<?>[] getRootApplicationContextInitializers() {
    return null;
}
```

在子类 `AbstractAnnotationConfigDispatcherServletInitializer` 中实现了 `createRootApplicationContext` 方法，如下：

```java
// AbstractAnnotationConfigDispatcherServletInitializer
@Override
@Nullable
protected WebApplicationContext createRootApplicationContext() {
    // 获取到 Root 配置的类
    Class<?>[] configClasses = getRootConfigClasses();
    if (!ObjectUtils.isEmpty(configClasses)) {
        // 如果配置的类不为空的话则新建一个 AnnotationConfigWebApplicationContext
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        // 将配置的类注册的 WebApplicationContext 中
        context.register(configClasses);
        return context;
    }
    else {
        return null;
    }
}

@Override
protected WebApplicationContext createServletApplicationContext() {
    // 创建 WebApplicationContext
    AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
    // 获取 Config 类
    Class<?>[] configClasses = getServletConfigClasses();
    if (!ObjectUtils.isEmpty(configClasses)) {
        // 注册 Config
        context.register(configClasses);
    }
    return context;
}
```



所以接下看一下 `ContextLoaderListener` 的 `contextInitialized` 方法，其如下：

```java
// ContextLoaderListener
@Override
public void contextInitialized(ServletContextEvent event) {
    // 初始化 Web 容器
    initWebApplicationContext(event.getServletContext());
}
```

```java
// ContextLoader
// 调用父类 ContextLoader 中的 initWebApplicationContext 初始化 Web 容器
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 判断是否已经初始化过了
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException(
            "Cannot initialize context because there is already a root application context present - " +
            "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }

    servletContext.log("Initializing Spring root WebApplicationContext");
    Log logger = LogFactory.getLog(ContextLoader.class);
    if (logger.isInfoEnabled()) {
        logger.info("Root WebApplicationContext: initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        // Store context in local instance variable, to guarantee that
        // it is available on ServletContext shutdown.
        // 如果没有 WebApplication 容器，则创建一个 WebApplicationContext 容器
        if (this.context == null) {
            this.context = createWebApplicationContext(servletContext);
        }
        if (this.context instanceof ConfigurableWebApplicationContext) {
            // 把获取到的 WebApplicationContext 转成 ConfigurableWebApplicationContext
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
            // 判断是否 Active 状态
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                // 如果当前 WebApplicationContext 的 ParentContext 为空
                // 则尝试获取 Parent 放到当前 Context 上
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent ->
                    // determine parent for root web application context, if any.
                    ApplicationContext parent = loadParentContext(servletContext);
                    cwac.setParent(parent);
                }
                // 配置并刷新容器
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }
        // 将 ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE 放到 Servlet 容器中
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

        // 存放 WebApplicationContext
        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
        if (ccl == ContextLoader.class.getClassLoader()) {
            currentContext = this.context;
        }
        else if (ccl != null) {
            currentContextPerThread.put(ccl, this.context);
        }

        if (logger.isInfoEnabled()) {
            long elapsedTime = System.currentTimeMillis() - startTime;
            logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
        }

        return this.context;
    }
    catch (RuntimeException | Error ex) {
        logger.error("Context initialization failed", ex);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
        throw ex;
    }
}

protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // 根据 ServletContext 获取 Context 的类
    Class<?> contextClass = determineContextClass(sc);
    // 如果获取到 Class 不是则抛错
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                                              "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    // 通过 BeanUtils 获取 WebApplicationContext
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}

protected Class<?> determineContextClass(ServletContext servletContext) {
    // 从 ServletContext 中获取初始化参数 contextClass 作为 contextClassName
    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
    if (contextClassName != null) {
        try {
            // 如果配置不为空则根据类名获取 Class 返回
            return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                "Failed to load custom context class [" + contextClassName + "]", ex);
        }
    }
    else {
        // 如果初始化参数中没有则从 ContextLoader.properties 配置文件中
        // 读取 WebApplicationContext 属性的值
        contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
        try {
            // 获取 Class 返回
            return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                "Failed to load default context class [" + contextClassName + "]", ex);
        }
    }
}

// 配置和刷新容器
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // The application context id is still set to its original default value
        // -> assign a more useful id based on available information
        // 为 WebApplicationContext 设置一个更有用 ID 基于现有的信息
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            // Generate default id...
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                      ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }

    // 将 ServletContext 放到 WebApplicationContext
    wac.setServletContext(sc);
    // 从 ServletContext 的初始化参数中读取 Config 的位置并放到 WebApplicationContext 上
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    // 获取当前的环境
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        // 设置初始化配置
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }
    // 自定义 WebApplicationContext
    customizeContext(sc, wac);
    // 调用 refresh 刷新容器，就是 Spring 的刷新流程
    wac.refresh();
}

protected void customizeContext(ServletContext sc, ConfigurableWebApplicationContext wac) {
    // 从 ServletContext 获取 ApplicationContextInitializer 类列表
    List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> initializerClasses =
        determineContextInitializerClasses(sc);
    // 遍历 ApplicationContextInitializer 列表
    for (Class<ApplicationContextInitializer<ConfigurableApplicationContext>> initializerClass : initializerClasses) {
        Class<?> initializerContextClass =
            GenericTypeResolver.resolveTypeArgument(initializerClass, ApplicationContextInitializer.class);
        if (initializerContextClass != null && !initializerContextClass.isInstance(wac)) {
            throw new ApplicationContextException(String.format(
                "Could not apply context initializer [%s] since its generic parameter [%s] " +
                "is not assignable from the type of application context used by this " +
                "context loader: [%s]", initializerClass.getName(), initializerContextClass.getName(),
                wac.getClass().getName()));
        }
        // 将排除掉不可以初始化的后的 ApplicationContextInitializer 
        // 实例化后放到  contextInitializers 
        this.contextInitializers.add(BeanUtils.instantiateClass(initializerClass));
    }

    // 对 contextInitializers 排序
    AnnotationAwareOrderComparator.sort(this.contextInitializers);
    // 遍历自定义的 ApplicationContextInitializer 逐个调用其 initialize 方法
    for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
        initializer.initialize(wac);
    }
}

// 从 ServletContext 中获取 ApplicationContextInitializer List
protected List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>>
			determineContextInitializerClasses(ServletContext servletContext) {

    List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> classes =
        new ArrayList<>();

    // 从 ServletContext 的初始化参数中获取 globalInitializerClasses 值
    String globalClassNames = servletContext.getInitParameter(GLOBAL_INITIALIZER_CLASSES_PARAM);
    if (globalClassNames != null) {
        for (String className : StringUtils.tokenizeToStringArray(globalClassNames, INIT_PARAM_DELIMITERS)) {
            classes.add(loadInitializerClass(className));
        }
    }

    // 从 ServletContext 的初始化参数中获取 contextInitializerClasses 的值
    String localClassNames = servletContext.getInitParameter(CONTEXT_INITIALIZER_CLASSES_PARAM);
    if (localClassNames != null) {
        for (String className : StringUtils.tokenizeToStringArray(localClassNames, INIT_PARAM_DELIMITERS)) {
            classes.add(loadInitializerClass(className));
        }
    }

    return classes;
}
```

然后我们回头到 `AbstractDispatcherServletInitializer` 中看起添加的一个步骤：

```java
// AbstractDispatcherServletInitializer
// 注册 Dispatcher Servlet
protected void registerDispatcherServlet(ServletContext servletContext) {
    // 获取 Servlet 的名称
    String servletName = getServletName();
    Assert.hasLength(servletName, "getServletName() must not return null or empty");

    // 创建 servletAppContext， Web IOC 容器
    WebApplicationContext servletAppContext = createServletApplicationContext();
    Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

    // 创建 dispatcherServlet
    FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
    Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
    dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

    // 在当前 ServletContext 中添加一个 dispatcherServlet
    ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
    if (registration == null) {
        throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
                                        "Check if there is another servlet registered under the same name.");
    }

    // 设置相关的属性信息
    // 加载时机
    registration.setLoadOnStartup(1);
    // 映射信息
    registration.addMapping(getServletMappings());
    // 异步支持
    registration.setAsyncSupported(isAsyncSupported());

    // 获取 Filter 并逐个注册到 ServletContext 中
    Filter[] filters = getServletFilters();
    if (!ObjectUtils.isEmpty(filters)) {
        for (Filter filter : filters) {
            registerServletFilter(servletContext, filter);
        }
    }

    // 留给子类的自定义方法
    customizeRegistration(registration);
}
```

到这里可以看到 Root 容器创建和刷新是在 `ContextLoaderListener` 的 `contextInitialized` 方法做的。Web 容器创建是在 `AbstractDispatcherServletInitializer` 中做的，但是并没加载器刷新容器的方法。通过 Debug 可知刷新容器是在 `HttpServletBean` 的 `init` 方法中调用的。如下所示：

 ```java
// HttpServletBean
@Override
public final void init() throws ServletException {

    // Set bean properties from init parameters.
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }

    // Let subclasses do whatever initialization they like.
    // 初始化 Servlet Bean
    initServletBean();
}
 ```

```java
// FrameworkServlet
@Override
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
    if (logger.isInfoEnabled()) {
        logger.info("Initializing Servlet '" + getServletName() + "'");
    }
    long startTime = System.currentTimeMillis();

    try {
        // 初始化 Web 容器
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (logger.isDebugEnabled()) {
        String value = this.enableLoggingRequestDetails ?
            "shown which may lead to unsafe logging of potentially sensitive data" :
        "masked to prevent unsafe logging of potentially sensitive data";
        logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                     "': request parameters and headers will be " + value);
    }

    if (logger.isInfoEnabled()) {
        logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
    }
}

// 初始化 Web 容器
protected WebApplicationContext initWebApplicationContext() {
    // 从 ServletContext 上获取 Root 容器
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    // 如果 Web 容器不为空则获取当前的 Web 容器
    if (this.webApplicationContext != null) {
        // A context instance was injected at construction time -> use it
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent -> set
                    // the root application context (if any; may be null) as the parent
                    // 设置 Web 容器的 Parent 为 Root 容器
                    cwac.setParent(rootContext);
                }
                // 配置和刷新 Web 容器
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    if (wac == null) {
        // No context instance was injected at construction time -> see if one
        // has been registered in the servlet context. If one exists, it is assumed
        // that the parent context (if any) has already been set and that the
        // user has performed any initialization such as setting the context id
        // 如果没有 Web 容器则尝试从 Servlet Context 中获取 Web 容器
        wac = findWebApplicationContext();
    }
    if (wac == null) {
        // No context instance is defined for this servlet -> create a local one
        // 如果还是没有根据 Root 容器创建一个 Web 容器并配置刷新容器
        wac = createWebApplicationContext(rootContext);
    }

    // 手动调用 容器刷新
    if (!this.refreshEventReceived) {
        // Either the context is not a ConfigurableApplicationContext with refresh
        // support or the context injected at construction time had already been
        // refreshed -> trigger initial onRefresh manually here.
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    // 将 Web 容器放到 Servlet Context 上
    if (this.publishContext) {
        // Publish the context as a servlet context attribute.
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}

protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    // 设置 ID
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // The application context id is still set to its original default value
        // -> assign a more useful id based on available information
        if (this.contextId != null) {
            wac.setId(this.contextId);
        }
        else {
            // Generate default id...
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                      ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
        }
    }

    // 存放 Servlet Context
    wac.setServletContext(getServletContext());
    // 存放 ServletConfig
    wac.setServletConfig(getServletConfig());
    // 存放 NameSpace
    wac.setNamespace(getNamespace());
    // 设置 Listener
    wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        // 初始化一些属性信息
        ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
    }

    // 留给子类的接口
    postProcessWebApplicationContext(wac);
    // 从 Servlet Context 中取出初始化参数配置单 Initializers 类
    // 并逐个调用其 initialize 方法
    applyInitializers(wac);
    // 调用容器的刷新过程
    wac.refresh();
}
```



通过以上的分析，清楚以注解的方法启动 SpringMVC 需要继承 `AbstractAnnotationConfigDispatcherServletInitializer` 类。实现其抽象方法指定 `DispatcherServlet` 的信息，同时指定两个容器分别管理 Web Bean 和 其他 Bean，如下图所示：

![Spring MVC 容器](http://img.sangzhenya.com/Snipaste_2019-11-30_20-26-43.png)

主要代码如下：

```java
public class MySpringMVCInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    // Root 容器 配置文件
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    // Web 容器 配置文件
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    // Servlet 的映射信息
    @Override
    protected String[] getServletMappings() {
        // "/" 拦截所有请求，不包括 *.jsp
        // "/*" 拦截所有请求，包含 *.jsp
        return new String[]{"/"};
    }
}
```

```java
@ComponentScan(value = "com.xinyue.myspring4", excludeFilters = {
    @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = {Controller.class})
})
public class RootConfig {
}
```

```java
@ComponentScan(value = "com.xinyue.myspring4", includeFilters = {
    @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = {Controller.class})
}, useDefaultFilters = false)
@EnableWebMvc // 开启 Spring MVC 定制
public class WebConfig implements WebMvcConfigurer {
    // 对于 Spring MVC 进行定制 等同 spring-mvc.xml config
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

```java
public class DemoController {
    private final DemoService demoService;

    public DemoController(DemoService demoService) {
        this.demoService = demoService;
    }

    @ResponseBody
    @RequestMapping("/demo")
    public String demo() {
        return demoService.demo() + "\n Demo From Controller";
    }
}
```

```java
@Service
public class DemoService {
    public String demo() {
        return "Demo From Service";
    }
}
```

测试结构如下：

![测试结果](http://img.sangzhenya.com/Snipaste_2019-11-30_22-57-26.png)





