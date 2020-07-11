---
title: "Spring MVC @RequestMapping 注解"
tags: ["Spring", "Spring MVC"]
categories: ["Spring"]
date: "2019-02-07T09:00:00+08:00"
---

SpringMVC 使用 @RequestMapping 注解为控制器指定可以处理哪些 URL 请求，在 Controller  的类和方法上都可以使用该注解，标记在类上提供初步的请求映射信息，相对于 Web 应用的根目录。标记在方法上提供进一步细分映射信息，对于标记在类上的 URL，如果方法所在的类未标注则  URL 是相对于 Web 应用的根目录。在 `DispatcherServlet` 截获请求后，就通过 @RequestMapping 提供的映射信息确定请求所对应的处理方法。RequestMapping  注解源码：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {
    // 设置拦截器的名称
	String name() default "";
    // 设置拦截的路径
	@AliasFor("path")
	String[] value() default {};
	@AliasFor("value")
	String[] path() default {};
    // 请求方式
	RequestMethod[] method() default {};
    // 拦截参数
	String[] params() default {};
    // 拦截请求头
	String[] headers() default {};
	String[] consumes() default {};
	String[] produces() default {};
}
```

一个简单的示例：

```java
@ResponseBody
@RequestMapping("/hello")
public String hello() {
    return "Hello World";
}
```

拦截指定请求方式的请求：

```java
@ResponseBody
@RequestMapping(value = "/method", method = RequestMethod.POST)
public String method() {
    return "Hello World";
}
```



```java
// name 必须是 1234 且 必须包含 password ，且不能包含 nickname
@ResponseBody
@RequestMapping(value = "/params", 
                params = {"name=1234", "password", "!nickname"})
public String params() {
    return "Hello World";
}
```

通过请求头参数过滤

```java
// 请求头中必须包含 Accept-Language
@ResponseBody
@RequestMapping(value = "/headers", headers = {"Accept-Language"})
public String headers() {
    return "Hello World";
}
```

也可以将参数拼接到 URL 中，使用 `@PathVariable` 注解

```java
@ResponseBody
@RequestMapping(value = "/customized/{id}")
public String customized(@PathVariable("id") int id) {
    return "Hello World " + id;
}
```



### Rest 架构风格

Rest  即 Representational  State Transfer 资源表现层状态转化，是比较流行的一种互联网软件架构。其结构清晰，符合标准，易于理解，所以在得到更多的网站。所谓资源 （Resource） 即网络上的一个实体，或者说网络上的一个具体信息。每个资源可以用一个 URI （统一资源定位符）指向它，每种资源对应一个特定的 URI。表现层 （Representation） 把资源具体呈现出来的形式，例如以 txt 程序或者网页格式呈现。状态转化即每发出一个请求，就代表了客户端和服务器的依次交互过程，HTTP 是无状态的，所以所有的状态都保存在服务端。因此如果客户端想要操作服务端必须通过某种手动让服务端发生状态转化。具体来说就是 HTTP 四种表示操作方式的动词 GET，POST，PUT，DELETE。分别对应 获取资源， 新建资源，更新资源和删除资源。

浏览器表单仅支持 GET 与 POST 请求。Spring 3.0 添加了一个过滤器（`HiddenHttpMethodFilter`），可以将 DELETE，PUT 方法转换为标准的 HTTP 方法。

过程如下：

因为 `HiddenHttpMethodFilter` 是一个 Filter 所以在访问到底 Servlet 之前会被执行，调用其 `doFilterInternal` 方法。

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
    throws ServletException, IOException {

    HttpServletRequest requestToUse = request;
	// 如果是 Post 请求且当前没有异常才进行转换
    if ("POST".equals(request.getMethod()) && request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) == null) {
        // 获取参数中的方法名称
        String paramValue = request.getParameter(this.methodParam);
        if (StringUtils.hasLength(paramValue)) {
            String method = paramValue.toUpperCase(Locale.ENGLISH);
            // 如果得到的方法名称不为空，且方法再运行的方法列表中（PUT,PATCH, DELETE）
            // 则对方法进行包装
            if (ALLOWED_METHODS.contains(method)) {
                requestToUse = new HttpMethodRequestWrapper(request, method);
            }
        }
    }
	// 调用 filter 链继续处理
    filterChain.doFilter(requestToUse, response);
}
```

包装的方法也是很简单：

```java
private static class HttpMethodRequestWrapper extends HttpServletRequestWrapper {

    private final String method;

    public HttpMethodRequestWrapper(HttpServletRequest request, String method) {
        super(request);
        // 就是把从参数中获取的方法放到包装对象上。
        this.method = method;
    }

    @Override
    public String getMethod() {
        return this.method;
    }
}
```










