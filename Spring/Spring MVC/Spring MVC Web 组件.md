---
title: "Spring MVC Web 组件"
tags: ["Spring", "Spring MVC"]
categories: ["Spring"]
date: "2019-02-13T09:00:00+08:00"
---

Web 主要有三个组件，分别是 servlet, filter, listener.

### Servlet

其主要作用是处理客户端请求的动态资源。Servlet 通常需要接受请求，处理请求，完成响应。其生命周期分为四个阶段，1 调用构造方法实例化，2 调用 init 方法初始化，3 处理请求调用 Service 方法，4 服务终止 调用 destory 方法。在 `web.xml` 配置一个简单的 Servlet 如下：

```xml
<servlet>
    <!-- Servlet 名称 -->
    <servlet-name>demo-servlet</servlet-name>
    <!-- 处理请求的类的全类名 -->
    <servlet-class>com.xinyue.myweb.MyServlet</servlet-class>
    <!-- 加载时机 -->
    <!-- 大于等于 0 表明启动即加载，值越小优先级越高。 -->
    <!-- 小于 0 或者为指定则在当前 servlet 被使用的时候加载 -->
    <load-on-startup>0</load-on-startup>
</servlet>
<servlet-mapping>
    <!-- 和 Servlet 名称保持一致 -->
    <servlet-name>demo-servlet</servlet-name>
    <url-pattern>/demo/*</url-pattern>
</servlet-mapping>
```

简单的  Servlet 类的实现如下：

```java
public class MyServlet extends HttpServlet {

    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.setCharacterEncoding("utf-8");
        resp.getWriter().write("Hello World");
    }
}
```

上面可以看出 doGet 方法中主要有两个参数 `HttpServletRequest` 和 `HttpServletResponse`。对于 `HttpServletRequest` 主要用于获取参数，头信息， cookie，请求地址等信息。而 `HttpServletResponse` 则用来设置返回头，添加 Cookie ，重定向，返回流等。

对于 Servlet 配置的类处理请求默认会调用 service 方法，在其父类中有默认实现，会根据 请求方式判断具体执行什么方法。默认实现如下：

```java
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String method = req.getMethod();
    long lastModified;
    if (method.equals("GET")) {
        lastModified = this.getLastModified(req);
        if (lastModified == -1L) {
            this.doGet(req, resp);
        } else {
            long ifModifiedSince;
            try {
                ifModifiedSince = req.getDateHeader("If-Modified-Since");
            } catch (IllegalArgumentException var9) {
                ifModifiedSince = -1L;
            }

            if (ifModifiedSince < lastModified / 1000L * 1000L) {
                this.maybeSetLastModified(resp, lastModified);
                this.doGet(req, resp);
            } else {
                resp.setStatus(304);
            }
        }
    } else if (method.equals("HEAD")) {
        lastModified = this.getLastModified(req);
        this.maybeSetLastModified(resp, lastModified);
        this.doHead(req, resp);
    } else if (method.equals("POST")) {
        this.doPost(req, resp);
    } else if (method.equals("PUT")) {
        this.doPut(req, resp);
    } else if (method.equals("DELETE")) {
        this.doDelete(req, resp);
    } else if (method.equals("OPTIONS")) {
        this.doOptions(req, resp);
    } else if (method.equals("TRACE")) {
        this.doTrace(req, resp);
    } else {
        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[]{method};
        errMsg = MessageFormat.format(errMsg, errArgs);
        resp.sendError(501, errMsg);
    }

}
```


### Filter

Filter 和 Servlet 有些相似，其主要负责过滤请求，在请求进入到 Servlet 之前判断是否满足限定条件，可以拦截进入 Servlet。其生命周期同样是四个阶段，1 调用构造器创建，2 调用 init 方法初始化，3 调用 doFilter 方法完成过滤功能，4 服务终止，调用 destory 方法。在 `web.xml` 中配置解决乱码的 filter 如下：

```xml
<!-- 解决乱码的问题 -->
<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <async-supported>true</async-supported>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
    <init-param>
        <param-name>forceEncoding</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

此外 Filter 默认只会拦截直接发送到目标资源的请求，而转发的请求不会拦截，如果需要拦截则需要手动配置拦截方式，总共有以下四种：
```xml
<dispatcher>REQUEST</dispatcher> 
<dispatcher>FORWARD</dispatcher>  
<dispatcher>INCLUDE</dispatcher>  
<dispatcher>ERROR</dispatcher>
```

### Listener

Listener 即监听器，可以用来监听Application、Session、Request对象，当这些对象发生变化就会调用对应的监听方法。在 `web.xml` 配置 Listener 如下：

```xml
<listener>
   <listener-class>com.xinyue.myWeb.MyListener</listener-class>
</listener>
```

主要可以分为三个 Scope 的监听：

1. ServletCotext

   `ServletContextListener` ：主要有两个方法  `contextInitialized` ，`contextDestroyed `。分别在创建 `ServletCotext` 的时候调用

   `ServletContextAttributeListener`：主要有三个方法 `attributeAdded`， `attributeReplaced`， `attributeRemoved`。分别在添加属性，修改属性和删除属性的时候调用。

2. HttpSession

   `HttpSessionListener` ：主要有两个方法  `sessionCreated` ，`sessionDestroyed`。分别在创建 `HttpSession` 的时候调用

   `HttpSessionAttributeListener`：主要有三个方法 `attributeAdded`， `attributeReplaced`， `attributeRemoved`。分别在添加属性，修改属性和删除属性的时候调用。

3. ServletRequest

   `ServletRequestListener` ：主要有两个方法  `requestInitialized` ，`requestDestroyed`。分别在创建 `ServletRequest` 的时候调用

   `ServletRequestAttributeListener`：主要有三个方法 `attributeAdded`， `attributeReplaced`， `attributeRemoved`。分别在添加属性，修改属性和删除属性的时候调用。

### 使用注解

此外如果使用 Servlet 3.0 及以上同样可以不用 `web.xml`, 而是直接通过注解的方法是注册三大组件。下面是一个简单通过注解实现 Servlet 的例子：
```java
// 使用@WebServlet 标注当前是一个 Servlet 
// 其值是需要拦截的 URL
@WebServlet("/hello")
public class MyServlet extends HttpServlet {
    // 服务端方法
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.getWriter().write("Hello World");
    }
}
```
同样的方法可以使用 `@WebFilter` 和 `@WebListener` 注册 Filter 和 Listener。

### 共享库和运行时插件
在 Servlet 3.0 中同时提供了一个机制是，Servlet 容器启动的时候会去扫描当前应用的每一个 Jar 包，查看类数据下 `META-INF/services/javax.servlet.ServletContainerInitializer` 文件中的内容，根据全类名找到所有 `ServletContainerInitializer` 子类并执行 `onStartup` 方法。
```java
// 感兴趣的类或接口
@HandlesTypes(value = {MyService.class})
public class MyServletContainerInitializer implements ServletContainerInitializer {
  // 参数 1 就是上面注解感兴趣类或接口的子接口或实现类的集合
    @Override
  public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        System.out.println("**************************************************");
        System.out.println(c);
        System.out.println("**************************************************");
    }
}
```
启动后打印日志如下：
```log
**************************************************
[class com.xinyue.myservlet.MyServiceImpl]
**************************************************
```

此外发现 `MyServletContainerInitializer` 中 `onStartup` 是 `ServletContext` 也就意味着可以在这个方法中动态的添加 Web 的三大组件。如下所示动态的为 Servlet Context 添加一个 Servlet。
```java
public class DemoServlet extends HttpServlet {
    @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("Demo World");
    }
}
```
在 `onStartup` 中添加。
```java
ServletRegistration.Dynamic demoServlet = ctx.addServlet("demo", new DemoServlet());
        demoServlet.addMapping("/demo");
```
当然在 `ServletContextListener` 的 `contextInitialized` 方法中也可以通过 `ServletContext servletContext = sce.getServletContext()` 获取到 `ServletContext`  进而动态的向 `ServletContext` 中添加组件。