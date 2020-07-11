---
title: "Spring MVC 自定义拦截器"
tags: ["Spring", "Spring MVC"]
categories: ["Spring"]
date: "2019-02-09T09:00:00+08:00"
---

Spring MVC 也可以使用拦截器对请求进行拦截处理，用户可以自定义拦截器来实现特定的功能，自定义的拦截器必须实现 `HandlerInterceptor` 接口，其主要有三个方法：

`preHandle()` 这个方法再业务处理请求之前被调用，在该方法中对用户请求 request 进行处理，如果需要该拦截器对请求进行拦截处理后还要调用其他拦截器或者业务方法进行处理则返回 true，否则返回 false。

`postHandle()` 这个方法再业务处理完请求之后，`DispatcherServlet` 向客户端返回响应之前被调用，在该方法中对用户请求 request 进行处理。

`afterCompletion()` 这个方法在 `DispatcherServlet` 完全处理完请求后调用，可以做一些资源清理的工作。

一个简单拦截器如下：

```java
public class MyHandlerInterceptor implements HandlerInterceptor {
    Log log = LogFactory.getLog(MyHandlerInterceptor.class);
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("preHandler");
        return true;
    }
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("postHandle");
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        log.info("afterCompletion");
    }
}
```

然后在 config 中添加该拦截器：

```java
 @Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new MyHandlerInterceptor()).addPathPatterns("/**");
}
```



Spring MVC 中的 Interceptor 执行流程如下所示：

主要的处理流程都在 `DispatcherServlet` 中的 `doDispatch` 方法中。

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            // 根据当前的请求信息获取到需要处理该请求的方法，调用其拦截器的 preHandle
            // 如果返回 false 则表示被拦截不继续进行
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            applyDefaultViewName(processedRequest, mv);
            // 在处理完请求之后调用拦截器的 postHandle 方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 出发处理 Handle 的结果，在最后一步会同样会调用 triggerAfterCompletion 方法
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        // 调用 complete 方法
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        // 调用 complete 方法
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

所以主要有三个地方

1. `if (!mappedHandler.applyPreHandle(processedRequest, response)) {`
2. `mappedHandler.applyPostHandle(processedRequest, response, mv);`
3. `triggerAfterCompletion(processedRequest, response, mappedHandler, ex);`

下面逐个来看其对应的方法：

首先是 `applyPreHandle` 方法

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 获取拦截该方法的所有拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 从 0 开始遍历拦截器逐个执行拦截器的方法
        for (int i = 0; i < interceptors.length; i++) {
            HandlerInterceptor interceptor = interceptors[i];
            // 调用拦截器的 preHandle 方法
            if (!interceptor.preHandle(request, response, this.handler)) {
                // 如果任何一个拦截器返回 False 则触发调用已经执行的拦截器的 afterCompletion 方法
                triggerAfterCompletion(request, response, null);
                return false;
            }
            // 记录成功执行了多少个拦截器的 preHandle
            this.interceptorIndex = i;
        }
    }
    return true;
}
```

然后再看 `applyPostHandle` 方法

```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
    throws Exception {
	
    // 同样是获取到所有的拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 倒序遍历拦截器逐个调用其 postHandle 方法
        for (int i = interceptors.length - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            interceptor.postHandle(request, response, this.handler, mv);
        }
    }
}
```

最后看一下 `triggerAfterCompletion` 方法

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
    throws Exception {
	
    // 同样是获取到所有的拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 遍历所有成功执行的所有的 preHandle 的拦截器
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            try {
                // 调用 拦截器的 afterCompletion 方法。
                interceptor.afterCompletion(request, response, this.handler, ex);
            }
            catch (Throwable ex2) {
                logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
            }
        }
    }
}
```

如果多个拦截器的时候会首先根据 Order 进行排序，同样 Order 的拦截器最先添加的最先执行。另外由上面的分析可知。优先级高的拦截器的 `preHandle` 方法先执行；之后优先级低的拦截器的的 `postHandle` 方法后执行；最后是执行所有已经执行了 `preHandle`  的拦截器的 `afterCompletion` 方法，优先级低的先执行。

