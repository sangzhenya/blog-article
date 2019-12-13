## Model And View



![ModelAndView](http://img.sangzhenya.com/Snipaste_2019-12-13_16-49-52.png)

Spring MVC 在 `ModelAndView` 上添加数据主要以下三个方法：

```java
public ModelAndView addObject(Object attributeValue) {
    // 调用添加数据的方法
    getModelMap().addAttribute(attributeValue);
    return this;
}
public ModelMap addAttribute(Object attributeValue) {
    Assert.notNull(attributeValue, "Model object must not be null");
    // 如果为空则直接返回
    if (attributeValue instanceof Collection && ((Collection<?>) attributeValue).isEmpty()) {
        return this;
    }
    // 根据数据类型获取类名作为参数名称
    return addAttribute(Conventions.getVariableName(attributeValue), attributeValue);
}
public ModelMap addAttribute(String attributeName, @Nullable Object attributeValue) {
    Assert.notNull(attributeName, "Model attribute name must not be null");
    // 根据属性名和属性值添加到 Map 中
    put(attributeName, attributeValue);
    return this;
}
public ModelAndView addObject(String attributeName, @Nullable Object attributeValue) {
    // 添加到 Map 中
    getModelMap().addAttribute(attributeName, attributeValue);
    return this;
}
public ModelAndView addAllObjects(@Nullable Map<String, ?> modelMap) {
    // 将传入的 Map 中所有的 Value 添加到 Map 中
    getModelMap().addAllAttributes(modelMap);
    return this;
}
public ModelMap addAllAttributes(@Nullable Map<String, ?> attributes) {
    if (attributes != null) {
        // 调用 Map 中 putAll 方法
        putAll(attributes);
    }
    return this;
}
```

通过 `setViewName` 设置 view 的名称

```java
public void setViewName(@Nullable String viewName) {
    this.view = viewName;
}
```

如果使用 `jsp` 需要在 web config 中配置 `config`：

```java
@Override
public void configureViewResolvers(ViewResolverRegistry registry) {
    registry.jsp("/WEB-INF/views/", ".jsp");
}
```

一个简单的 `ModelAndView` 使用如下：

```java
@RequestMapping("/view")
public ModelAndView demo() {
    ModelAndView mv = new ModelAndView();
    mv.addObject("key1", "value1");
    mv.setViewName("view");
    return mv;
}
```

同样也可以使用以下两种方式向 Model 中存放数据：

```java
@RequestMapping("/view1")
public String demo1(Map<String, Object> modelMap) {
    modelMap.put("key1", "value1");
    return "view";
}
@RequestMapping("/view2")
public String demo2(Model model) {
    model.addAttribute("key1", "value1");
    return "view";
}
```

这两种方法都会将数据放到 Model 中。下面看一下 业务方法调用及视图处理的流程：

主要是 `DispatcherServlet` 中的 `doDispatch` 方法中的

1.  `mv = ha.handle(processedRequest, response, mappedHandler.getHandler());` 调用业务处理方法。
2.  `processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);` 解析视图。

首先看一下 `handle` 方法：

```java
// AbstractHandlerMethodAdapter
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
    throws Exception {
	// 直接调用 handleInternal 方法
    return handleInternal(request, response, (HandlerMethod) handler);
}
```

```java
// RequestMappingHandlerAdapter
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
                                      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ModelAndView mav;
    // 检查请求方法和Session 是否正确
    checkRequest(request);

    // 是否需要根据 Session 同步
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            // 获取到 Session 则根据 Session 获取 mutex 对其加锁
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            // 拿不到就直接执行
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    }
    else {
        // 拿不到就直接执行
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }

    // 如果 response 不包含 HEADER_CACHE_CONTROL
    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        // 如果通过 handlerMethod 获取 Session 属性有 Session 标签则设置cache 的时长
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        }
        else {
            // 否则就准备返回的 Response
            prepareResponse(response);
        }
    }

    return mav;
}
```

```java
// RequestMappingHandlerAdapter
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                           HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	// 将 request 和 response 封装到 ServletWebRequest 中
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        // 获取数据绑定工厂
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        // 获取 Model 工厂
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        // 创建包装 handlerMethod
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            // 如果参数不为空给包装对象设置 resolvers
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            // 如果返回值不为空给包装对象设置 returnValueHandlers
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        // 给包装方法设置 数据绑定工厂
        invocableMethod.setDataBinderFactory(binderFactory);
        // 设置 parameterNameDiscoverer
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        // 创建一个 ModelAndViewContainer
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        // 为 Model 赋值
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        // 将 request 和 response 封装到 AsyncWebRequest 中
        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
		// 设置异步超时时间        
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        // 同 Request 中获取 WebAsyncManager 如果没有则创建一个
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        // 设置 TaskExecutor
        asyncManager.setTaskExecutor(this.taskExecutor);
        // 设置 request 和 response 封装的 AsyncWebRequest
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        // 注册异步的 拦截器
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        // 如果已经有了结果
        if (asyncManager.hasConcurrentResult()) {
            // 获取结果
            Object result = asyncManager.getConcurrentResult();
            // 获取 ModelAndView Container
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            // 清除结果
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            //包装结果
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }

        // 调用处理逻辑
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }

        // 获取 Model And View
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

```java
// RequestMappingHandlerAdapter
private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
    Class<?> handlerType = handlerMethod.getBeanType();
    // 根据方法的所在的类从 Cache 中获取标记有 InitBinder 的 method set
    Set<Method> methods = this.initBinderCache.get(handlerType);
    if (methods == null) {
        // 如果返回 null 则从类中去查找对应的方法并放到 Cache 中
        methods = MethodIntrospector.selectMethods(handlerType, INIT_BINDER_METHODS);
        this.initBinderCache.put(handlerType, methods);
    }
    // 初始化一个 initBinderMethods
    List<InvocableHandlerMethod> initBinderMethods = new ArrayList<>();
    // 拿全局的 initBinderAdvice cache 根据 handleType 遍历过滤放到 initBinderMethods 中
    this.initBinderAdviceCache.forEach((clazz, methodSet) -> {
        if (clazz.isApplicableToBeanType(handlerType)) {
            Object bean = clazz.resolveBean();
            for (Method method : methodSet) {
                initBinderMethods.add(createInitBinderMethod(bean, method));
            }
        }
    });
    // 遍历上面拿到 methods 放到 initBinderMethods 中
    for (Method method : methods) {
        Object bean = handlerMethod.getBean();
        initBinderMethods.add(createInitBinderMethod(bean, method));
    }
    // 创建 ServletRequestDataBinderFactory
    return createDataBinderFactory(initBinderMethods);
}
```

```java
// RequestMappingHandlerAdapter
private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
    // 根据 Handle Method 获取 SessionAttributesHandler
    SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);
    Class<?> handlerType = handlerMethod.getBeanType();
    // 根据 handlerType 从 modelAttributeCache 获取 ModelAttribute Method Set
    Set<Method> methods = this.modelAttributeCache.get(handlerType);
    if (methods == null) {
        methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
        this.modelAttributeCache.put(handlerType, methods);
    }
    // 初始化 attrMethods
    List<InvocableHandlerMethod> attrMethods = new ArrayList<>();
    // 拿全局的 ModelAttribute Method Set
    this.modelAttributeAdviceCache.forEach((clazz, methodSet) -> {
        if (clazz.isApplicableToBeanType(handlerType)) {
            Object bean = clazz.resolveBean();
            for (Method method : methodSet) {
                attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
            }
        }
    });
    for (Method method : methods) {
        Object bean = handlerMethod.getBean();
        attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
    }
    // 创建 ModelFactory
    return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
}
```

```java
// ServletInvocableHandlerMethod
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
                            Object... providedArgs) throws Exception {
	// 调用业务方法
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    // 设置返回状态
    setResponseStatus(webRequest);

    if (returnValue == null) {
        if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
            disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return;
        }
    }
    else if (StringUtils.hasText(getResponseStatusReason())) {
        mavContainer.setRequestHandled(true);
        return;
    }

    mavContainer.setRequestHandled(false);
    Assert.state(this.returnValueHandlers != null, "No return value handlers");
    try {
        // 使用 returnValueHandlers 处理 returnValue
        this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
    catch (Exception ex) {
        if (logger.isTraceEnabled()) {
            logger.trace(formatErrorForReturnValue(returnValue), ex);
        }
        throw ex;
    }
}
```

```java
// InvocableHandlerMethod
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
                               Object... providedArgs) throws Exception {

    // 获取方法参数
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }
    // 根据生成的参数值调用方法
    return doInvoke(args);
}
```

```java
// InvocableHandlerMethod
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

    // 获取方法参数
    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }

    // 创建调用业务方法需要的参数的数组
    Object[] args = new Object[parameters.length];
    // 遍历参数列表
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        // 先从传入的 providedArgs 中获取参数值
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
        }
        try {
            // 使用 HandlerMethodArgumentResolverComposite 获取生成参数值
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
        }
        catch (Exception ex) {
            // Leave stack trace for later, exception may actually be resolved and handled...
            if (logger.isDebugEnabled()) {
                String exMsg = ex.getMessage();
                if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                    logger.debug(formatArgumentError(parameter, exMsg));
                }
            }
            throw ex;
        }
    }
    return args;
}
```

```java
// HandlerMethodArgumentResolverComposite
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

    // 根据方法参数获取 HandlerMethodArgumentResolver
    HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
    if (resolver == null) {
        throw new IllegalArgumentException("Unsupported parameter type [" +
                                           parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
    }
    // 调用上面获取的 resolver 去获取参数值
    // 如果是 Map 使用 MapMethodProcessor，如果是 Model 使用 ModelMethodProcessor
    // 两者均直接从 ModelAndView Container 中获取 Model 对象 mavContainer.getModel()
    // 同样也有其他各种从参数中获取值的 resolver
    return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

```java
// HandlerMethodArgumentResolverComposite
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    // 首先从 Cache 中获取
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        // 获取不到就遍历方法参数 argumentResolvers 尝试获取
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            // 判断是否支持，支持就返回
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}
```

```java
// HandlerMethodReturnValueHandlerComposite
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	// 根据 returnType 获取相应的 HandlerMethodReturnValueHandler
    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    }
    // 处理 Return Value 其中一个是 ViewNameMethodReturnValueHandler
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

```java
// ViewNameMethodReturnValueHandler
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue instanceof CharSequence) {
        String viewName = returnValue.toString();
        // 如果是字节类型则设置 viewName 到 ModelAndViewContainer 上
        mavContainer.setViewName(viewName);
        // 如果 view name 是 redirect 则 设置 RedirectModelScenario 为 true
        // 判断方法是 (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
        if (isRedirectViewName(viewName)) {
            mavContainer.setRedirectModelScenario(true);
        }
    }
    else if (returnValue != null) {
        // should not happen
        throw new UnsupportedOperationException("Unexpected return type: " +
                                                returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
    }
}
```

```java
// RequestMappingHandlerAdapter
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
                                     ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

    // 更新 Model
    modelFactory.updateModel(webRequest, mavContainer);
    if (mavContainer.isRequestHandled()) {
        return null;
    }
    // 从 Model And View Container 中的 Model
    ModelMap model = mavContainer.getModel();
    // 根据 ModelAndView Container 中的生成新的一个 Model And View
    ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
    if (!mavContainer.isViewReference()) {
        mav.setView((View) mavContainer.getView());
    }
    if (model instanceof RedirectAttributes) {
        Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        if (request != null) {
            RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
        }
    }
    // 返回生成的 ModelAndView
    return mav;
}
```

```java
public void updateModel(NativeWebRequest request, ModelAndViewContainer container) throws Exception {
	// 获取 Default ModelMap
    ModelMap defaultModel = container.getDefaultModel();
    if (container.getSessionStatus().isComplete()){
        this.sessionAttributesHandler.cleanupAttributes(request);
    }
    else {
        // 存储一部分的属性到 Session 中
        this.sessionAttributesHandler.storeAttributes(request, defaultModel);
    }
    // 更新绑定的结果
    if (!container.isRequestHandled() && container.getModel() == defaultModel) {
        updateBindingResult(request, defaultModel);
    }
}
```

上面就是调用业务方法的流程。

接下来看一下 `processDispatchResult` 方法。

```java
// DispatcherServlet
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

    boolean errorView = false;

    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }

    // Did the handler return a view to render?
    if (mv != null && !mv.wasCleared()) {
        // render view
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isTraceEnabled()) {
            logger.trace("No view rendering, null ModelAndView returned.");
        }
    }

    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        // Concurrent handling started during a forward
        return;
    }

    if (mappedHandler != null) {
        // Exception (if any) is already handled..
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}
```

```java
// DispatcherServlet
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    // Determine locale for request and apply it to the response.
    Locale locale =
        (this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
    // 设置 locale
    response.setLocale(locale);

    View view;
    // 获取 view name
    String viewName = mv.getViewName();
    if (viewName != null) {
        // 解析视图
        view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
                                       "' in servlet with name '" + getServletName() + "'");
        }
    }
    else {
        // No need to lookup: the ModelAndView object contains the actual View object.
        view = mv.getView();
        if (view == null) {
            throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
                                       "View object in servlet with name '" + getServletName() + "'");
        }
    }

    // Delegate to the View object for rendering.
    if (logger.isTraceEnabled()) {
        logger.trace("Rendering view [" + view + "] ");
    }
    try {
        if (mv.getStatus() != null) {
            response.setStatus(mv.getStatus().value());
        }
        // 根据获取到的 view 调用其 render 方法
        view.render(mv.getModelInternal(), request, response);
    }
    catch (Exception ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Error rendering view [" + view + "]", ex);
        }
        throw ex;
    }
}
```

```java
// // DispatcherServlet
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
                               Locale locale, HttpServletRequest request) throws Exception {

    // 遍历 viewResolvers 尝试解析视图
    if (this.viewResolvers != null) {
        for (ViewResolver viewResolver : this.viewResolvers) {
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
                return view;
            }
        }
    }
    return null;
}
```

```java
// AbstractView
public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
                   HttpServletResponse response) throws Exception {

    if (logger.isDebugEnabled()) {
        logger.debug("View " + formatViewName() +
                     ", model " + (model != null ? model : Collections.emptyMap()) +
                     (this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
    }

    // 创建 Merge Output Model
    Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
    // 准备 Response
    prepareResponse(request, response);
    // Render 内部资源给予的特定 model，设置 model 到 request 的属性上
    renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
}
```

```java
protected void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

    // Expose the model object as request attributes.
    // 将 Model 上的属性暴露到 request 上
    exposeModelAsRequestAttributes(model, request);

    // Expose helpers as request attributes, if any.
    exposeHelpers(request);

    // Determine the path for the request dispatcher.
    // 确定 request dispatcher 的路径
    String dispatcherPath = prepareForRendering(request, response);

    // Obtain a RequestDispatcher for the target resource (typically a JSP).
    // 获取 RequestDispatcher
    RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
    if (rd == null) {
        throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
                                   "]: Check that the corresponding file exists within your web application archive!");
    }

    // If already included or response already committed, perform include, else forward.
    if (useInclude(request, response)) {
        response.setContentType(getContentType());
        if (logger.isDebugEnabled()) {
            logger.debug("Including [" + getUrl() + "]");
        }
        rd.include(request, response);
    }

    else {
        // Note: The forwarded resource is supposed to determine the content type itself.
        if (logger.isDebugEnabled()) {
            logger.debug("Forwarding to [" + getUrl() + "]");
        }
        // 转发请求
        rd.forward(request, response);
    }
}
```

### Callable 方法

一个简单的 Callable 的 异步处理 方法如下：

```java
@ResponseBody
@RequestMapping(value = "/mvcasync1")
public Callable<String> myAsync() {
    log.info("Main Start");
    Callable<String> callable = () -> {
        log.info("Sub Start");
        Thread.sleep(3000);
        log.info("Sub End");
        return "Demo String";
    };
    log.info("Main End");
    return callable;
}
```

主要流程的差别是是使用不同的 `HandlerMethodReturnValueHandler` 处理结果，对于 `Callable ` 使用 `CallableMethodReturnValueHandler` 处理结果。主要方法如下：

```java
// CallableMethodReturnValueHandler
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue == null) {
        mavContainer.setRequestHandled(true);
        return;
    }

    Callable<?> callable = (Callable<?>) returnValue;
    // 启动异步处理
    WebAsyncUtils.getAsyncManager(webRequest).startCallableProcessing(callable, mavContainer);
}
```

```java
// WebAsyncManager
public void startCallableProcessing(Callable<?> callable, Object... processingContext) throws Exception {
    Assert.notNull(callable, "Callable must not be null");
    startCallableProcessing(new WebAsyncTask(callable), processingContext);
}
```

```java
// WebAsyncManager
public void startCallableProcessing(final WebAsyncTask<?> webAsyncTask, Object... processingContext)
			throws Exception {

    Assert.notNull(webAsyncTask, "WebAsyncTask must not be null");
    Assert.state(this.asyncWebRequest != null, "AsyncWebRequest must not be null");

    // 获取超时时间
    Long timeout = webAsyncTask.getTimeout();
    if (timeout != null) {
        this.asyncWebRequest.setTimeout(timeout);
    }

    // 获取 AsyncTaskExecutor
    AsyncTaskExecutor executor = webAsyncTask.getExecutor();
    if (executor != null) {
        this.taskExecutor = executor;
    }
    else {
        logExecutorWarning();
    }

    // 获取异步处理的 拦截器
    List<CallableProcessingInterceptor> interceptors = new ArrayList<>();
    interceptors.add(webAsyncTask.getInterceptor());
    interceptors.addAll(this.callableInterceptors.values());
    interceptors.add(timeoutCallableInterceptor);

    // 获取 Callable 
    final Callable<?> callable = webAsyncTask.getCallable();
    // 获取 Callable 的拦截链
    final CallableInterceptorChain interceptorChain = new CallableInterceptorChain(interceptors);

    // 添加超时处理器
    this.asyncWebRequest.addTimeoutHandler(() -> {
        if (logger.isDebugEnabled()) {
            logger.debug("Async request timeout for " + formatRequestUri());
        }
		// 触发超时处理
        Object result = interceptorChain.triggerAfterTimeout(this.asyncWebRequest, callable);
        if (result != CallableProcessingInterceptor.RESULT_NONE) {
            setConcurrentResultAndDispatch(result);
        }
    });

    // 添加错误处理器
    this.asyncWebRequest.addErrorHandler(ex -> {
        if (logger.isDebugEnabled()) {
            logger.debug("Async request error for " + formatRequestUri() + ": " + ex);
        }
        // 触发错误处理
        Object result = interceptorChain.triggerAfterError(this.asyncWebRequest, callable, ex);
        result = (result != CallableProcessingInterceptor.RESULT_NONE ? result : ex);
        setConcurrentResultAndDispatch(result);
    });

    // 添加完成处理器
    this.asyncWebRequest.addCompletionHandler(() ->
                                              interceptorChain.triggerAfterCompletion(this.asyncWebRequest, callable));

    // 触发 拦截器的 beforeConcurrentHandling 
    interceptorChain.applyBeforeConcurrentHandling(this.asyncWebRequest, callable);
    // 开启异步处理
    startAsyncProcessing(processingContext);
    try {
        // 提交 Task
        Future<?> future = this.taskExecutor.submit(() -> {
            Object result = null;
            try {
                // 调用拦截器的 preProcess
                interceptorChain.applyPreProcess(this.asyncWebRequest, callable);
                // 调用业务异步业务处理方法
                result = callable.call();
            }
            catch (Throwable ex) {
                result = ex;
            }
            finally {
                // 调用拦截器的 postProcess
                result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, result);
            }
            // 获取同步结果并返回
            setConcurrentResultAndDispatch(result);
        });
        // 将 future 放到拦截器链中
        interceptorChain.setTaskFuture(future);
    }
    catch (RejectedExecutionException ex) {
        Object result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, ex);
        setConcurrentResultAndDispatch(result);
        throw ex;
    }
}
```

### DeferredResult 方法

一个简单的 DeferredResult 方法如下：

```java
@ResponseBody
@RequestMapping(value = "/mvcasync2")
public DeferredResult<Object> myAsync2() {
    DeferredResult<Object> deferredResult = new DeferredResult<>(10000L, "121231");
    DeferredResultQueue.save(deferredResult);
    return deferredResult;
}

@ResponseBody
@RequestMapping("/mvcasync3")
public String myAsync3() {
    DeferredResult<Object> objectDeferredResult = DeferredResultQueue.get();
    UUID uuid = UUID.randomUUID();
    objectDeferredResult.setResult(uuid.toString());
    return "My Async3:::" + uuid;
}
```

与上面的同样只是 `RetrunValueHandler ` 不同，其使用 `DeferredResultMethodReturnValueHandler` 处理 `ReturnValue`。

```java
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue == null) {
        mavContainer.setRequestHandled(true);
        return;
    }

    DeferredResult<?> result;

    // 包装 value
    if (returnValue instanceof DeferredResult) {
        result = (DeferredResult<?>) returnValue;
    }
    else if (returnValue instanceof ListenableFuture) {
        result = adaptListenableFuture((ListenableFuture<?>) returnValue);
    }
    else if (returnValue instanceof CompletionStage) {
        result = adaptCompletionStage((CompletionStage<?>) returnValue);
    }
    else {
        // Should not happen...
        throw new IllegalStateException("Unexpected return value type: " + returnValue);
    }

    // 启动起步处理
    WebAsyncUtils.getAsyncManager(webRequest).startDeferredResultProcessing(result, mavContainer);
}

private DeferredResult<Object> adaptListenableFuture(ListenableFuture<?> future) {
    DeferredResult<Object> result = new DeferredResult<>();
    future.addCallback(new ListenableFutureCallback<Object>() {
        @Override
        public void onSuccess(@Nullable Object value) {
            result.setResult(value);
        }
        @Override
        public void onFailure(Throwable ex) {
            result.setErrorResult(ex);
        }
    });
    return result;
}

private DeferredResult<Object> adaptCompletionStage(CompletionStage<?> future) {
    DeferredResult<Object> result = new DeferredResult<>();
    future.handle((BiFunction<Object, Throwable, Object>) (value, ex) -> {
        if (ex != null) {
            if (ex instanceof CompletionException && ex.getCause() != null) {
                ex = ex.getCause();
            }
            result.setErrorResult(ex);
        }
        else {
            result.setResult(value);
        }
        return null;
    });
    return result;
}
```

```java
// 与处理 Callable 方法类似
public void startDeferredResultProcessing(
			final DeferredResult<?> deferredResult, Object... processingContext) throws Exception {

    Assert.notNull(deferredResult, "DeferredResult must not be null");
    Assert.state(this.asyncWebRequest != null, "AsyncWebRequest must not be null");

    Long timeout = deferredResult.getTimeoutValue();
    if (timeout != null) {
        this.asyncWebRequest.setTimeout(timeout);
    }

    List<DeferredResultProcessingInterceptor> interceptors = new ArrayList<>();
    interceptors.add(deferredResult.getInterceptor());
    interceptors.addAll(this.deferredResultInterceptors.values());
    interceptors.add(timeoutDeferredResultInterceptor);

    final DeferredResultInterceptorChain interceptorChain = new DeferredResultInterceptorChain(interceptors);

    this.asyncWebRequest.addTimeoutHandler(() -> {
        try {
            interceptorChain.triggerAfterTimeout(this.asyncWebRequest, deferredResult);
        }
        catch (Throwable ex) {
            setConcurrentResultAndDispatch(ex);
        }
    });

    this.asyncWebRequest.addErrorHandler(ex -> {
        try {
            if (!interceptorChain.triggerAfterError(this.asyncWebRequest, deferredResult, ex)) {
                return;
            }
            deferredResult.setErrorResult(ex);
        }
        catch (Throwable interceptorEx) {
            setConcurrentResultAndDispatch(interceptorEx);
        }
    });

    this.asyncWebRequest.addCompletionHandler(()
                                              -> interceptorChain.triggerAfterCompletion(this.asyncWebRequest, deferredResult));

    interceptorChain.applyBeforeConcurrentHandling(this.asyncWebRequest, deferredResult);
    startAsyncProcessing(processingContext);

    try {
        interceptorChain.applyPreProcess(this.asyncWebRequest, deferredResult);
        // 设置 Result Handler
        deferredResult.setResultHandler(result -> {
            result = interceptorChain.applyPostProcess(this.asyncWebRequest, deferredResult, result);
            // 获取结果并处理
            setConcurrentResultAndDispatch(result);
        });
    }
    catch (Throwable ex) {
        setConcurrentResultAndDispatch(ex);
    }
}

private void startAsyncProcessing(Object[] processingContext) {
    synchronized (WebAsyncManager.this) {
        this.concurrentResult = RESULT_NONE;
        this.concurrentResultContext = processingContext;
    }
    this.asyncWebRequest.startAsync();

    if (logger.isDebugEnabled()) {
        logger.debug("Started async request");
    }
}

// 设置结果
public boolean setResult(T result) {
    return setResultInternal(result);
}

private boolean setResultInternal(Object result) {
    // Immediate expiration check outside of the result lock
    if (isSetOrExpired()) {
        return false;
    }
    // 获取 Result Handle 并执行
    DeferredResultHandler resultHandlerToUse;
    synchronized (this) {
        // Got the lock in the meantime: double-check expiration status
        if (isSetOrExpired()) {
            return false;
        }
        // At this point, we got a new result to process
        this.result = result;
        resultHandlerToUse = this.resultHandler;
        if (resultHandlerToUse == null) {
            // No result handler set yet -> let the setResultHandler implementation
            // pick up the result object and invoke the result handler for it.
            return true;
        }
        // Result handler available -> let's clear the stored reference since
        // we don't need it anymore.
        this.resultHandler = null;
    }
    // If we get here, we need to process an existing result object immediately.
    // The decision is made within the result lock; just the handle call outside
    // of it, avoiding any deadlock potential with Servlet container locks.
    // Handle Result
    resultHandlerToUse.handleResult(result);
    return true;
}
```







