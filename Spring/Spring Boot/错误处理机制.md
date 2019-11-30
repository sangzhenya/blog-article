## Spring Boot 错误处理机制

Spring 针对于浏览器访问和客户端访问使用不同的处理逻辑。如果是浏览器则返回一个出错的网页，而如果是客户端则返回 JSON 格式的错误数据。其处理的原理如下：

在 `ErrorMvcAutoConfiguration` 有对于出错处理的自动配置。其为容器中添加了 `DefaultErrorAttributes` Bean 在页面中共享信息。

```java
@Bean
@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
public DefaultErrorAttributes errorAttributes() {
    return new DefaultErrorAttributes(this.serverProperties.getError().isIncludeException());
}
```

此外为容器中添加了 `BasicErrorController` Bean，处理 error 的请求

```java
@Bean
@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
public BasicErrorController basicErrorController(ErrorAttributes errorAttributes,
                                                 ObjectProvider<ErrorViewResolver> errorViewResolvers) {
    return new BasicErrorController(errorAttributes, this.serverProperties.getError(),
                                    errorViewResolvers.orderedStream().collect(Collectors.toList()));
}
```

此外为容器中添加了 `ErrorPageCustomizer` Bean。

```java
@Bean
public ErrorPageCustomizer errorPageCustomizer(DispatcherServletPath dispatcherServletPath) {
    return new ErrorPageCustomizer(this.serverProperties, dispatcherServletPath);
}
```

此外为容器添加了 `DefaultErrorViewResolver` Bean，默认的错误视图解析器

```java
@Bean
@ConditionalOnBean(DispatcherServlet.class)
@ConditionalOnMissingBean(ErrorViewResolver.class)
DefaultErrorViewResolver conventionErrorViewResolver() {
    return new DefaultErrorViewResolver(this.applicationContext, this.resourceProperties);
}
```



当系统出现了 4xx 或 5xx 之类的错误的时候，`ErrorPageCustomizer` 就会生效定制出错页面出错规则，然后会转发到 `/error` 请求中，被 `BasicErrorController` 处理。响应页面是由 `DefaultErrorViewResolver` 解析生成的。
```java
// 浏览器请求
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
    HttpStatus status = getStatus(request);
    Map<String, Object> model = Collections
        .unmodifiableMap(getErrorAttributes(request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
    response.setStatus(status.value());
    ModelAndView modelAndView = resolveErrorView(request, response, status, model);
    return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
}

// 客户端请求
@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
    HttpStatus status = getStatus(request);
    if (status == HttpStatus.NO_CONTENT) {
        return new ResponseEntity<>(status);
    }
    Map<String, Object> body = getErrorAttributes(request, isIncludeStackTrace(request, MediaType.ALL));
    return new ResponseEntity<>(body, status);
}
```

在有模板引擎的情况下，可是将 错误页面命名为 错误码.html 放到模板引擎的 error 文件夹下，发生对应的状态码的错误就会来到对应的页面。页面可以获取 `timestamp` 时间戳，`status` 状态码，`error` 错误提示，`exception` 异常对象，`message` 异常消息，`errors` JSR303 数据校验的错误等信息。

如果没有模板引擎，则会在静态资源文件夹找。

如果都没有则使用 Spring Boot 的默认提示页面。

```java
@Override
public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
    ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
    if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
        modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
    }
    return modelAndView;
}

// 找模板引擎的文件
private ModelAndView resolve(String viewName, Map<String, Object> model) {
    String errorViewName = "error/" + viewName;
    TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName,
                                                                                           this.applicationContext);
    if (provider != null) {
        return new ModelAndView(errorViewName, model);
    }
    return resolveResource(errorViewName, model);
}

// 找静态资源文件夹中的文件
private ModelAndView resolveResource(String viewName, Map<String, Object> model) {
    for (String location : this.resourceProperties.getStaticLocations()) {
        try {
            Resource resource = this.applicationContext.getResource(location);
            resource = resource.createRelative(viewName + ".html");
            if (resource.exists()) {
                return new ModelAndView(new HtmlResourceView(resource), model);
            }
        }
        catch (Exception ex) {
        }
    }
    return null;
}
```

如果想要对于异常处理同样也可以使用自定义异常处理器实现。如下所示：

```java
// 定制数据返回
@ControllerAdvice
public class MyErrHandler {
    @ResponseBody
    @ExceptionHandler(Exception.class)
    public Map<String, Object> handleException(Exception ex) {
        Map<String, Object> dataMap = new HashMap<>();
        dataMap.put("code", "XXX");
        dataMap.put("message", ex.getMessage());
        return dataMap;
    }
}
// 使用默认的处理方式
@ControllerAdvice
public class MyErrHandler {
    @ExceptionHandler(Exception.class)
    public String handleExceptionByDefault(Exception ex, HttpServletRequest request) {
        // 注意设置 status code 否则不会进入错误处理
        request.setAttribute("javax.servlet.error.status_code",500);
        return "forward:/error";
    }
}
```

如果需要定制数据则需要使用 `ErrorController`, 出现错误后会来到 `error`  被 `BasicErrorController`  而相应出去的数据是由 `getErrorAttributes` 得到的。所以可以通过编写 `ErrorController` 或 `AbstractErrorController` 类放到容器中进行数据的定制。如下所示：

```java
@Component
public class MyErrAttributes extends DefaultErrorAttributes {
    @Override
    public Map<String, Object> getErrorAttributes(WebRequest webRequest, boolean includeStackTrace) {
        Map<String, Object> errorAttributes = super.getErrorAttributes(webRequest, includeStackTrace);
        errorAttributes.put("customized", "Customized Data");
        return errorAttributes;
    }
}
```

