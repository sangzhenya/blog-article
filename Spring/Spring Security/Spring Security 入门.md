---
title: "Spring Security 入门"
tags: ["Spring", "Spring Security"]
categories: ["Spring"]
date: "2019-02-15T09:00:00+08:00"
---

Spring Security 就是通过一条过滤器链使用认证和授权的工作。如下图所示：

![](http://img.programya.com/20200201140026.png)

其中 FilterSecurityInterceptor 中会根据 config 配置的内容，判断是否可以访问对应的资源，如果不可以则会抛出异常。然后在 ExceptionTranslationFilter 中会捕获异常并作出响应的处理。

Spring Security 几个重要的概念如下：

1. SecurityContextHolder

   Spring Security 提供的 SecurityContextHolder 类让程序可以方便访问 SecurityContext。其采用的是 ThreadLocal 方式存储，保证获取到的永远是当前用户的 SecurityContext。

2. SecurityContext

   SecurityContext 保存用户的信息，用户是谁，权限有哪些，是否被认证等。可以通过 SecurityContextHolder.getContext() 获取。

3. Authentication

   Authentication 包含 sessionId, IP, 用户的 UserDetail 信息，用户的角色。其有很多实现类，每种类都对应着一个认证方式。可以通过 SecurityContextHolder.getContext().getAuthentication(); 获取。

4. AbstractAuthenticationToken

   AbstractAuthenticationToken 和 AnonymousAuthenticationToken 及其很多子类都实现了 Authentication 接口，每一个子类代表一种认证方式。

5. GenericFilterBean

   用来拦截认证的 filter，可以在这个 filter 注册认证成功和失败的 handler，子类主要有 OncePerRequestFilter, CasAuthenticationFilter, OpenIDAuthenticationFilter, UsernamePasswordAuthenticationFilter, LogoutFilter 等等。

6. AuthenticationManager 和 AuthenticationProvider

   Spring  Security 将通过 SecurityFilter 来调用 AuthenticationManager 进行验证，默认情况下 AuthenticationManager 将验证工作交给 ProviderManager 去做，而 ProviderManager 会通过一系列的 AuthenticationProvider 完成具体的验证。

7. UserDetails 和 UserDetailsService

   UserDetail 包含 Authentication 需要的用户信息，这些信息往往从 JDBC 或其他数据源获取。UserDetailsService 是用来获取指定 username 对应 UserDetail 的接口。

AbstractAuthenticationProcessingFilter 部分源代码如下：

````java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
  HttpServletRequest request = (HttpServletRequest) req;
  HttpServletResponse response = (HttpServletResponse) res;
	// 是否是登录的链接，默认是 login， POST 方法
  if (!requiresAuthentication(request, response)) {
    // 如果不是则直接继续接下来的 filter
    chain.doFilter(request, response);
    return;
  }

  if (logger.isDebugEnabled()) {
    logger.debug("Request is to process authentication");
  }

  Authentication authResult;
  try {
    // 认证并获取结果
    authResult = attemptAuthentication(request, response);
    if (authResult == null) {
      // return immediately as subclass has indicated that it hasn't completed
      // authentication
      return;
    }
    sessionStrategy.onAuthentication(authResult, request, response);
  }
  catch (InternalAuthenticationServiceException failed) {
    logger.error(
      "An internal error occurred while trying to authenticate the user.",
      failed);
    unsuccessfulAuthentication(request, response, failed);

    return;
  }
  catch (AuthenticationException failed) {
    // Authentication failed
    unsuccessfulAuthentication(request, response, failed);

    return;
  }

  // Authentication success
  if (continueChainBeforeSuccessfulAuthentication) {
    chain.doFilter(request, response);
  }

  successfulAuthentication(request, response, chain, authResult);
}
````



UsernamePasswordAuthenticationFilter 部分源代码如下：

```java
// UsernamePasswordAuthenticationFilter
public UsernamePasswordAuthenticationFilter() {
  // 该 Filter 仅处理 /login 的 post 请求
  super(new AntPathRequestMatcher("/login", "POST"));
}

// 认证方法如下
public Authentication attemptAuthentication(HttpServletRequest request,
                                            HttpServletResponse response) throws AuthenticationException {
  if (postOnly && !request.getMethod().equals("POST")) {
    throw new AuthenticationServiceException(
      "Authentication method not supported: " + request.getMethod());
  }

  // 获取用户名和密码
  String username = obtainUsername(request);
  String password = obtainPassword(request);

  if (username == null) {
    username = "";
  }

  if (password == null) {
    password = "";
  }

  username = username.trim();

  // 根据 username 和 password 生成 token
  UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
    username, password);

  // Allow subclasses to set the "details" property
  setDetails(request, authRequest);

  // 认证并返回结果
  return this.getAuthenticationManager().authenticate(authRequest);
}
```

ExceptionTranslationFilter 部分源代码如下

```java
// ExceptionTranslationFilter
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
  HttpServletRequest request = (HttpServletRequest) req;
  HttpServletResponse response = (HttpServletResponse) res;

  try {
    // 直接做 filter 主要处理逻辑在 catch 中
    chain.doFilter(request, response);
    logger.debug("Chain processed normally");
  }
  catch (IOException ex) {
    throw ex;
  }
  catch (Exception ex) {
    // Try to extract a SpringSecurityException from the stacktrace
    Throwable[] causeChain = throwableAnalyzer.determineCauseChain(ex);
    // 获取 AuthenticationException
    RuntimeException ase = (AuthenticationException) throwableAnalyzer
      .getFirstThrowableOfType(AuthenticationException.class, causeChain);
    if (ase == null) {
      ase = (AccessDeniedException) throwableAnalyzer.getFirstThrowableOfType(
        AccessDeniedException.class, causeChain);
    }

    if (ase != null) {
      if (response.isCommitted()) {
        throw new ServletException("Unable to handle the Spring Security Exception because the response is already committed.", ex);
      }
      // 处理验证失败
      handleSpringSecurityException(request, response, chain, ase);
    }
    else {
      // Rethrow ServletExceptions and RuntimeExceptions as-is
      if (ex instanceof ServletException) {
        throw (ServletException) ex;
      }
      else if (ex instanceof RuntimeException) {
        throw (RuntimeException) ex;
      }

      // Wrap other Exceptions. This shouldn't actually happen
      // as we've already covered all the possibilities for doFilter
      throw new RuntimeException(ex);
    }
  }
}
// 根据不同错误类型做出不同的处理
private void handleSpringSecurityException(HttpServletRequest request,
                                           HttpServletResponse response, FilterChain chain, RuntimeException exception)
  throws IOException, ServletException {
  if (exception instanceof AuthenticationException) {
    logger.debug(
      "Authentication exception occurred; redirecting to authentication entry point",
      exception);

    sendStartAuthentication(request, response, chain,
                            (AuthenticationException) exception);
  }
  else if (exception instanceof AccessDeniedException) {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    if (authenticationTrustResolver.isAnonymous(authentication) || authenticationTrustResolver.isRememberMe(authentication)) {
      logger.debug(
        "Access is denied (user is " + (authenticationTrustResolver.isAnonymous(authentication) ? "anonymous" : "not fully authenticated") + "); redirecting to authentication entry point",
        exception);

      sendStartAuthentication(
        request,
        response,
        chain,
        new InsufficientAuthenticationException(
          messages.getMessage(
            "ExceptionTranslationFilter.insufficientAuthentication",
            "Full authentication is required to access this resource")));
    }
    else {
      logger.debug(
        "Access is denied (user is not anonymous); delegating to AccessDeniedHandler",
        exception);

      accessDeniedHandler.handle(request, response,
                                 (AccessDeniedException) exception);
    }
  }
}
```

FilterSecurityInterceptor 部分源代码如下：

```java
public void doFilter(ServletRequest request, ServletResponse response,
                     FilterChain chain) throws IOException, ServletException {
  FilterInvocation fi = new FilterInvocation(request, response, chain);
  invoke(fi);
}

public void invoke(FilterInvocation fi) throws IOException, ServletException {
  if ((fi.getRequest() != null)
      && (fi.getRequest().getAttribute(FILTER_APPLIED) != null)
      && observeOncePerRequest) {
    // filter already applied to this request and user wants us to observe
    // once-per-request handling, so don't re-do security checking
    fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
  } else {
    // first time this request being called, so perform security checking
    if (fi.getRequest() != null && observeOncePerRequest) {
      fi.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
    }

    // 判断是否可以请求资源
    InterceptorStatusToken token = super.beforeInvocation(fi);

    try {
      fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
    } finally {
      super.finallyInvocation(token);
    }

    super.afterInvocation(token, null);
  }
}
```

AbstractSecurityInterceptor 部分源码如下：

```java
protected InterceptorStatusToken beforeInvocation(Object object) {
  Assert.notNull(object, "Object was null");
  final boolean debug = logger.isDebugEnabled();

  if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
    throw new IllegalArgumentException(
      "Security invocation attempted for object "
      + object.getClass().getName()
      + " but AbstractSecurityInterceptor only configured to support secure objects of type: "
      + getSecureObjectClass());
  }

  // 获取当前 URL 的属性信息
  Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
    .getAttributes(object);

  if (attributes == null || attributes.isEmpty()) {
    if (rejectPublicInvocations) {
      throw new IllegalArgumentException(
        "Secure object invocation "
        + object
        + " was denied as public invocations are not allowed via this interceptor. "
        + "This indicates a configuration error because the "
        + "rejectPublicInvocations property is set to 'true'");
    }

    if (debug) {
      logger.debug("Public object - authentication not attempted");
    }

    publishEvent(new PublicInvocationEvent(object));

    return null; // no further work post-invocation
  }

  if (debug) {
    logger.debug("Secure object: " + object + "; Attributes: " + attributes);
  }

  // 判断是否有 Authentication
  if (SecurityContextHolder.getContext().getAuthentication() == null) {
    credentialsNotFound(messages.getMessage(
      "AbstractSecurityInterceptor.authenticationNotFound",
      "An Authentication object was not found in the SecurityContext"),
                        object, attributes);
  }

  // 获取 Authentication
  Authentication authenticated = authenticateIfRequired();

  // Attempt authorization
  try {
    this.accessDecisionManager.decide(authenticated, object, attributes);
  }
  catch (AccessDeniedException accessDeniedException) {
    // 发布验证失败的事件
    publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,
                                               accessDeniedException));
		// 抛出异常
    throw accessDeniedException;
  }

  if (debug) {
    logger.debug("Authorization successful");
  }

  if (publishAuthorizationSuccess) {
    // 发布验证成功的事件
    publishEvent(new AuthorizedEvent(object, attributes, authenticated));
  }

  // Attempt to run as a different user
  Authentication runAs = this.runAsManager.buildRunAs(authenticated, object,
                                                      attributes);
  if (runAs == null) {
    if (debug) {
      logger.debug("RunAsManager did not change Authentication object");
    }

    // no further work post-invocation
    return new InterceptorStatusToken(SecurityContextHolder.getContext(), false,
                                      attributes, object);
  }
  else {
    if (debug) {
      logger.debug("Switching to RunAs Authentication: " + runAs);
    }

    SecurityContext origCtx = SecurityContextHolder.getContext();
    SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
    SecurityContextHolder.getContext().setAuthentication(runAs);

    // need to revert to token.Authenticated post-invocation
    return new InterceptorStatusToken(origCtx, true, attributes, object);
  }
}

private Authentication authenticateIfRequired() {
  // 从 Context 中获取 Authentication
  Authentication authentication = SecurityContextHolder.getContext()
    .getAuthentication();

  // 如果已经有 Authentication 则直接返回
  if (authentication.isAuthenticated() && !alwaysReauthenticate) {
    if (logger.isDebugEnabled()) {
      logger.debug("Previously Authenticated: " + authentication);
    }
    return authentication;
  }

  // 使用 authenticationManager 进行验证获取 authentication
  authentication = authenticationManager.authenticate(authentication);

  // We don't authenticated.setAuthentication(true), because each provider should do
  // that
  if (logger.isDebugEnabled()) {
    logger.debug("Successfully Authenticated: " + authentication);
  }

  // 将获得的 authentication 放在 Context
  SecurityContextHolder.getContext().setAuthentication(authentication);
  return authentication;
}
```

AffirmativeBased 部分源码如下：

```java
public void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {
  int deny = 0;

  // 遍历 voters
  for (AccessDecisionVoter voter : getDecisionVoters()) {
    // 调用 voter 的 vote 方法并获取结果
    int result = voter.vote(authentication, object, configAttributes);

    if (logger.isDebugEnabled()) {
      logger.debug("Voter: " + voter + ", returned: " + result);
    }

    switch (result) {
      case AccessDecisionVoter.ACCESS_GRANTED:
        return;

      case AccessDecisionVoter.ACCESS_DENIED:
        deny++;

        break;

      default:
        break;
    }
  }

  if (deny > 0) {
    // 如果 deny 则抛出异常
    throw new AccessDeniedException(messages.getMessage(
      "AbstractAccessDecisionManager.accessDenied", "Access is denied"));
  }

  // To get this far, every AccessDecisionVoter abstained
  checkAllowIfAllAbstainDecisions();
}
```

ProviderManager 部分源码如下

```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
  Class<? extends Authentication> toTest = authentication.getClass();
  AuthenticationException lastException = null;
  AuthenticationException parentException = null;
  Authentication result = null;
  Authentication parentResult = null;
  boolean debug = logger.isDebugEnabled();

  // 遍历 AuthenticationProvider
  for (AuthenticationProvider provider : getProviders()) {
    // 支持当前类型才继续
    if (!provider.supports(toTest)) {
      continue;
    }

    if (debug) {
      logger.debug("Authentication attempt using "
                   + provider.getClass().getName());
    }

    try {
      // 调用 provider 的方法去验证
      result = provider.authenticate(authentication);

      if (result != null) {
        copyDetails(authentication, result);
        break;
      }
    }
    catch (AccountStatusException e) {
      prepareException(e, authentication);
      // SEC-546: Avoid polling additional providers if auth failure is due to
      // invalid account status
      throw e;
    }
    catch (InternalAuthenticationServiceException e) {
      prepareException(e, authentication);
      throw e;
    }
    catch (AuthenticationException e) {
      lastException = e;
    }
  }

  if (result == null && parent != null) {
    // Allow the parent to try.
    try {
      // 若干当前 providerManager 中没有则尝试使用其父类去找
      result = parentResult = parent.authenticate(authentication);
    }
    catch (ProviderNotFoundException e) {
      // ignore as we will throw below if no other exception occurred prior to
      // calling parent and the parent
      // may throw ProviderNotFound even though a provider in the child already
      // handled the request
    }
    catch (AuthenticationException e) {
      lastException = parentException = e;
    }
  }

  if (result != null) {
    if (eraseCredentialsAfterAuthentication
        && (result instanceof CredentialsContainer)) {
      // Authentication is complete. Remove credentials and other secret data
      // from authentication
      ((CredentialsContainer) result).eraseCredentials();
    }

    // If the parent AuthenticationManager was attempted and successful than it will publish an AuthenticationSuccessEvent
    // This check prevents a duplicate AuthenticationSuccessEvent if the parent AuthenticationManager already published it
    if (parentResult == null) {
      eventPublisher.publishAuthenticationSuccess(result);
    }
    return result;
  }

  // Parent was null, or didn't authenticate (or throw an exception).

  if (lastException == null) {
    lastException = new ProviderNotFoundException(messages.getMessage(
      "ProviderManager.providerNotFound",
      new Object[] { toTest.getName() },
      "No AuthenticationProvider found for {0}"));
  }

  // If the parent AuthenticationManager was attempted and failed than it will publish an AbstractAuthenticationFailureEvent
  // This check prevents a duplicate AbstractAuthenticationFailureEvent if the parent AuthenticationManager already published it
  if (parentException == null) {
    prepareException(lastException, authentication);
  }

  throw lastException;
}
```

AbstractUserDetailsAuthenticationProvider 部分源代码如下

```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
  Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
                      messages.getMessage(
                        "AbstractUserDetailsAuthenticationProvider.onlySupports",
                        "Only UsernamePasswordAuthenticationToken is supported"));

  // Determine username
  String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
    : authentication.getName();

  boolean cacheWasUsed = true;
  // 首先从缓存中获取
  UserDetails user = this.userCache.getUserFromCache(username);

  if (user == null) {
    cacheWasUsed = false;

    try {
      // 获取用户
      user = retrieveUser(username,
                          (UsernamePasswordAuthenticationToken) authentication);
    }
    catch (UsernameNotFoundException notFound) {
      logger.debug("User '" + username + "' not found");

      if (hideUserNotFoundExceptions) {
        throw new BadCredentialsException(messages.getMessage(
          "AbstractUserDetailsAuthenticationProvider.badCredentials",
          "Bad credentials"));
      }
      else {
        throw notFound;
      }
    }

    Assert.notNull(user,
                   "retrieveUser returned null - a violation of the interface contract");
  }

  try {
    // 对用户是否过期一类校验
    preAuthenticationChecks.check(user);
    // 校验用户密码等信息
    additionalAuthenticationChecks(user,
                                   (UsernamePasswordAuthenticationToken) authentication);
  }
  catch (AuthenticationException exception) {
    if (cacheWasUsed) {
      // There was a problem, so try again after checking
      // we're using latest data (i.e. not from the cache)
      cacheWasUsed = false;
      user = retrieveUser(username,
                          (UsernamePasswordAuthenticationToken) authentication);
      preAuthenticationChecks.check(user);
      additionalAuthenticationChecks(user,
                                     (UsernamePasswordAuthenticationToken) authentication);
    }
    else {
      throw exception;
    }
  }

  postAuthenticationChecks.check(user);

  if (!cacheWasUsed) {
    this.userCache.putUserInCache(user);
  }

  Object principalToReturn = user;

  if (forcePrincipalAsString) {
    principalToReturn = user.getUsername();
  }

  return createSuccessAuthentication(principalToReturn, authentication, user);
}
```

DaoAuthenticationProvider 部分源码如下

```java
protected final UserDetails retrieveUser(String username,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
  prepareTimingAttackProtection();
  try {
    // 使用 UserDetail Service 获取 User 
    UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
    if (loadedUser == null) {
      throw new InternalAuthenticationServiceException(
        "UserDetailsService returned null, which is an interface contract violation");
    }
    return loadedUser;
  }
  catch (UsernameNotFoundException ex) {
    mitigateAgainstTimingAttack(authentication);
    throw ex;
  }
  catch (InternalAuthenticationServiceException ex) {
    throw ex;
  }
  catch (Exception ex) {
    throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
  }
}
```



