---
title: "Spring MVC 运行流程"
tags: ["Spring", "Spring MVC"]
categories: ["Spring"]
date: "2019-02-05T09:00:00+08:00"
---

调用的起点是 `org.springframework.web.servlet.DispatcherServlet#doService` 方法

```java
// DispatcherServlet
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);

    // Keep a snapshot of the request attributes in case of an include,
    // to be able to restore the original attributes after the include.
    Map<String, Object> attributesSnapshot = null;
    if (WebUtils.isIncludeRequest(request)) {
        attributesSnapshot = new HashMap<>();
        Enumeration<?> attrNames = request.getAttributeNames();
        while (attrNames.hasMoreElements()) {
            String attrName = (String) attrNames.nextElement();
            if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                attributesSnapshot.put(attrName, request.getAttribute(attrName));
            }
        }
    }

    // Make framework objects available to handlers and view objects.
    // 在 request 中设置一些属性
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

    if (this.flashMapManager != null) {
        FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
        }
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
    }

    try {
        // 开始派发
        doDispatch(request, response);
    }
    finally {
        if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            // Restore the original attribute snapshot, in case of an include.
            if (attributesSnapshot != null) {
                restoreAttributesAfterInclude(request, attributesSnapshot);
            }
        }
    }
}
```

```java
// DispatcherServlet
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    // 从 request 中获取 WebAsyncManager
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 检查是否是多部分请求
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            // 从 Request 中获取 HandlerExecutionChain 执行链
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                // 对于没有找到 Handler 的处理
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            // 获取 Handler 的适配器
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            // 获取 request 方法
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            // 逐个调用 拦截器的 preHandle
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            // 真正去调用 Handler (参考 Model And View)
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            // 如果是并发的则直接 return
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            // 如果没有 mv 则设置 default view
            applyDefaultViewName(processedRequest, mv);
            // 调用拦截器的 postHandle 方法
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
        // 处理分发的结果 (参考 Model And View)
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        // 触发拦截器的 afterCompletion
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        // 触发拦截器的 afterCompletion
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                // 调用异步拦截器的方法
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

protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // 遍历 handlerMappings
        for (HandlerMapping mapping : this.handlerMappings) {
            // 从 HandlerMapping 中获取 handler
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                // 返回执行链 
                return handler;
            }
        }
    }
    return null;
}

protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
    if (pageNotFoundLogger.isWarnEnabled()) {
        pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
    }
    if (this.throwExceptionIfNoHandlerFound) {
        throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
                                          new ServletServerHttpRequest(request).getHeaders());
    }
    else {
        // 转发到 404
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
    }
}

protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    // 遍历适配器
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                // 返回适配器
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
                               "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

```java
// AbstractHandlerMethodMapping
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 根据 Request 获取 Handler
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        handler = getDefaultHandler();
    }
    if (handler == null) {
        return null;
    }
    // Bean name or resolved handler?
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }

    // 根据 handler 和 request 获取 HandlerExecutionChain
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

    if (logger.isTraceEnabled()) {
        logger.trace("Mapped to " + handler);
    }
    else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
        logger.debug("Mapped to " + executionChain.getHandler());
    }

    //针配置有 Cors 的 handler 处理
    if (hasCorsConfigurationSource(handler)) {
        CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        config = (config != null ? config.combine(handlerConfig) : handlerConfig);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }

    // 返回执行链
    return executionChain;
}
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    // 获取 path
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    // 放到请求上
    request.setAttribute(LOOKUP_PATH, lookupPath);
    // 加锁
    this.mappingRegistry.acquireReadLock();
    try {
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
        //  释放锁
        this.mappingRegistry.releaseReadLock();
    }
}
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    // Handler 执行链
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
                                   (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

    // 获取路径
    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
    // 遍历拦截器
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        if (interceptor instanceof MappedInterceptor) {
            // 如果是 MappedInterceptor 在判断后加入到拦截器链中
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        }
        else {
            // 否则之间加入拦截器链中
            chain.addInterceptor(interceptor);
        }
    }
    return chain;
}
```

```java
// AbstractHandlerMethodMapping
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
    List<Match> matches = new ArrayList<>();
    // 根据 Url 获取  patchMatch
    List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
    if (directPathMatches != null) {
        // 放到 matches 中
        addMatchingMappings(directPathMatches, matches, request);
    }
    if (matches.isEmpty()) {
        // No choice but to go through all mappings...
        addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
    }

    if (!matches.isEmpty()) {
        // matches 排序，根据 method condition，patternsCondition，paramsCondition 等
        Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
        matches.sort(comparator);
        // 获取最 match 的
        Match bestMatch = matches.get(0);
        if (matches.size() > 1) {
            if (logger.isTraceEnabled()) {
                logger.trace(matches.size() + " matching mappings: " + matches);
            }
            if (CorsUtils.isPreFlightRequest(request)) {
                return PREFLIGHT_AMBIGUOUS_MATCH;
            }
            Match secondBestMatch = matches.get(1);
            // 如果有两个 method 可以 match 当前 url 则抛出异常
            if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                Method m1 = bestMatch.handlerMethod.getMethod();
                Method m2 = secondBestMatch.handlerMethod.getMethod();
                String uri = request.getRequestURI();
                throw new IllegalStateException(
                    "Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
            }
        }
        // 将 handlerMethod 放到 request 上
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
        handleMatch(bestMatch.mapping, lookupPath, request);
        return bestMatch.handlerMethod;
    }
    else {
        return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
    }
}
```

```java
// AbstractHandlerMethodMapping
private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
    // 遍历 mapping
    for (T mapping : mappings) {
        // 从 mapping 中获取对应 request 的 match
        T match = getMatchingMapping(mapping, request);
        if (match != null) {
            // 将获取到的 match 加上 HandlerMethod 放到 matches
            matches.add(new Match(match, this.mappingRegistry.getMappings().get(mapping)));
        }
    }
}
protected RequestMappingInfo getMatchingMapping(RequestMappingInfo info, HttpServletRequest request) {
    return info.getMatchingCondition(request);
}
```

```java
// RequestMappingInfo
public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
    RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
    if (methods == null) {
        return null;
    }
    ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
    if (params == null) {
        return null;
    }
    HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
    if (headers == null) {
        return null;
    }
    ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
    if (consumes == null) {
        return null;
    }
    ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
    if (produces == null) {
        return null;
    }
    PatternsRequestCondition patterns = this.patternsCondition.getMatchingCondition(request);
    if (patterns == null) {
        return null;
    }
    RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
    if (custom == null) {
        return null;
    }

    // 上面简单来讲就是根据不同的过滤条件过滤方法，把过滤的结果封装成一个 RequestMappingInfo
    return new RequestMappingInfo(this.name, patterns,
                                  methods, params, headers, consumes, produces, custom.getCondition());
}
```

以上是 Spring MVC 处理请求的主要流程。













