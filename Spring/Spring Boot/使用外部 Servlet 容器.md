## 使用外部容器

### 基本项目创建

在创建项目的时候使用 war 的方式打包：

![项目创建](http://img.programya.com/Snipaste_2019-12-18_22-13-39.png)

其实主要的 pom 文件的配置如下：

```xml
<groupId>com.example</groupId>
<artifactId>webwar</artifactId>
<version>0.0.1-SNAPSHOT</version>
<!--打包方式设置为 war-->
<packaging>war</packaging>

<!--将内置的 Tomcat 设置为 provided -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-tomcat</artifactId>
  <scope>provided</scope>
</dependency>
```

在主目录下编写 `SpringBootServletInitializer` 的子类并实现其 `configure` 方法，样例如下：

```java
public class ServletInitializer extends SpringBootServletInitializer {
	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		// 将 主类作为参数传入
    return application.sources(WebwarApplication.class);
	}
}
```



在项目设置中设置 Web Source 的目录

![Web Source](http://img.programya.com/Snipaste_2019-12-18_22-19-06.png)

然后设置 Edit Configuration

![设置 Tomcat](http://img.programya.com/Snipaste_2019-12-18_22-21-18.png)

在 `src/main/webapp/`添加 `index.jsp` 文件，然后启动就可以在浏览器中看到成功的页面。

![成功页面](http://img.programya.com/Snipaste_2019-12-18_22-24-36.png)

在配置中配置 view 相关的信息

```properties
spring.mvc.view.prefix=/WEB-INF/
spring.mvc.view.suffix=.jsp
```

然后在容器中添加对应 Controller 即可

```java
@GetMapping("/hello")
public String index() {
  return "hello";
}
```



### 外部容器的启动原理

eb 容器启动的时候会扫描每个 jar 包下的 `META-INF\services\javax.servlet.ServletContainerInitializer` 文件中的内容，根据文件中的定义全类名找到 `ServletContainerInitializer` 的子类，然后调用其 `onStartup` 方法。在引入的 `spring-web` 下就有这样的一个文件，其内容是 `org.springframework.web.SpringServletContainerInitializer`， 即定义了一个 `SpringServletContainerInitializer`， 容器启动的时候会执行该类的 `onStartup` 方法，在其中会拿到所有的 `WebApplicationInitializer` 的子类逐个调用其 `onStartup` 方法。

我们自定义的 `ServletInitializer` 最终就是实现了 `WebApplicationInitializer` 接口。主要的逻辑在其父类 `SpringBootServletInitializer` 中。如下所示：

```java
// SpringBootServletInitializer
// 容器启动的时候会被调用到
public void onStartup(ServletContext servletContext) throws ServletException {
  // Logger initialization is deferred in case an ordered
  // LogServletContextInitializer is being used
  this.logger = LogFactory.getLog(getClass());
  // 创建一个 根 Application 容器
  WebApplicationContext rootAppContext = createRootApplicationContext(servletContext);
  if (rootAppContext != null) {
    servletContext.addListener(new ContextLoaderListener(rootAppContext) {
      @Override
      public void contextInitialized(ServletContextEvent event) {
        // no-op because the application context is already initialized
      }
    });
  }
  else {
    this.logger.debug("No ContextLoaderListener registered, as createRootApplicationContext() did not "
                      + "return an application context");
  }
}

protected WebApplicationContext createRootApplicationContext(ServletContext servletContext) {
  // 创建 Spring Application Builder
  SpringApplicationBuilder builder = createSpringApplicationBuilder();
  builder.main(getClass());
  // 尝试获取父容器，如果获取到说明已经创建一个 Root Application Context 直接使用就好了
  ApplicationContext parent = getExistingRootWebApplicationContext(servletContext);
  if (parent != null) {
    this.logger.info("Root context already created (using as parent).");
    servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, null);
    builder.initializers(new ParentContextApplicationContextInitializer(parent));
  }
  // 设置初始化器
  builder.initializers(new ServletContextApplicationContextInitializer(servletContext));
  // 设置 ApplicationContext 类型
  builder.contextClass(AnnotationConfigServletWebServerApplicationContext.class);
  // config builder 是一个留给子类的接口
  // 我们就是实现了这个接口并设置把我们的主类作为 source 设置到了 builder 上
  builder = configure(builder);
  // 设置 listener
  builder.listeners(new WebEnvironmentPropertySourceInitializer(servletContext));
  // Build 生成一个 Spring Application
  SpringApplication application = builder.build();
  if (application.getAllSources().isEmpty()
      && MergedAnnotations.from(getClass(), SearchStrategy.TYPE_HIERARCHY).isPresent(Configuration.class)) {
    application.addPrimarySources(Collections.singleton(getClass()));
  }
  Assert.state(!application.getAllSources().isEmpty(),
               "No SpringApplication sources have been defined. Either override the "
               + "configure method or add an @Configuration annotation");
  // Ensure error pages are registered
  if (this.registerErrorPageFilter) {
    application.addPrimarySources(Collections.singleton(ErrorPageFilterConfiguration.class));
  }
  // 启动 Spring 容器
  // 内部直接调用 (WebApplicationContext) application.run() 
  // 和我们通过主类中 SpringApplication.run(WebwarApplication.class, args); 
  // 最终调用的代码是一致的
  return run(application);
}
protected SpringApplicationBuilder createSpringApplicationBuilder() {
  return new SpringApplicationBuilder();
}
```

以上就是启动 Tomcat，Spring 容器的初始化过程。