---
title: "Spring Security"
tags: ["Spring", "Spring Security"]
categories: ["Spring"]
date: "2020-02-15T09:00:00+08:00"
---

[toc]

## 权限相关概念

权限管理一般是值根据系统设置的安全规则或者安全策略，用户只能访问被授权的资源。权限管理的前置条件是用户和密码的认证系统。其中认证 是通过用户名和密码成功登陆系统后，让系统得到当前用户的角色身份；授权 则是系统根据当前用户的角色，给其授予可以操作的的权限资源。

主要有三个对象：

1. 用户：用户名，密码和当前用户的角色信息，可以实现认证操作
2. 角色：角色名称，描述和当前角色拥有的权限信息，可以实现授权操作。
3. 权限：权限也可以成为菜单，主要包含当前权限的名称，URL 地址等信息，可以动态展示菜单。

## Spring Security 基本概念

Spring Security 是 Spring 采用 AOP 思想，基于 Servlet 过滤器实现的安全框架，其提供了完善的认证机制和方法级授权。其主要是通过过滤器实现认证和授权的，可以简单看一下 Spring Security 中的过滤器。

### Spring 主要的 Filters

1. `SecurityContextPersistenceFilter`

   `SecurityContextPersistenceFilter` 主要是使用 `SecurityContextRepository` 在 Session 中保存或更新一个 `SecurityContext`，并将 `SecurityContext` 给后续的过滤器中使用，来为后续 filter 建立所需的上下文，`SecurityContext` 中存储了当前用户认证及授权信息。

2. `WebAsyncManagerIntegrationFilter`

   用于集成 `SecurityContext` 到 Spring 异步执行的 `WebAsyncManager` 

3. `HeaderWriterFilter`

   向请求头 Header 中添加相应的信息。

4. `CsrfFilter`

   Spring Security 会对所有 POST 请求验证是否包含系统生成的 CSRF 的 Token 信息，如果不包含则报错，起到防止 CSRF 共计的效果。

5. `LogoutFilter`

   匹配 URL 为 /logout 的请求，实现用户退出清除认证信息。

6. `UsernamePasswordAuthenticationFilter`

   认证操作，默认匹配 URL  为 /login 且必须为 POST 请求

7. `DefaultLoginPageGeneratingFilter`

   默认的登录页面，该过滤器会生成一个默认的认证页面

8. `DefaultLogoutPageGeneratingFilter`

   默认的登出页面，该过滤器会生成一个默认的登出页面。

9. `BasicAuthenticationFilter`

   过滤器会自动解析 HTTP 请求头中名字为 Authentication，且以 BASIC 开头的头部信息

   > Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==

10. `RequestCacheAwareFilter`

    通过 `HttpSessionRequestCache` 内部维护一个 RequestCache，用于缓存 HttpServletRequest

11. `SecurityContextHolderAwareRequestFilter`

    针对于 ServletRequest 进行一次包装，使 Request 具有更多的 API。

12. `AnonymousAuthenticationFilter`

    在 SecurityContextHolder 中认证信息为空的时候，会创建一个匿名的用户存入到 SecurityContextHolder 中，Spring Security 为了兼容未登录的访问，使用匿名的身份走了一套认证的流程。

13. `SessionManagementFilter`

    SecurityContextRepository 限制一个用户开启多个会话的数量。

14. `ExceptionTranslationFilter`

    转换过滤器链中出现的异常，在过滤器链的最后。

15. `FilterSecurityInterceptor`

    获取所配置资源的访问授权信息，根据 SecurityContextHolder 中存储的用户信息决定是否有权限访问。

### Spring Security 加载及应用 Filter 流程

在 Spring Security 项目启动的时候会走到 `WebSecurityConfigurerAdapter` 的 `getHttp` 方法中，该方法中部分代码如下：

```java
http
  .csrf().and()
  .addFilter(new WebAsyncManagerIntegrationFilter())
  .exceptionHandling().and()
  .headers().and()
  .sessionManagement().and()
  .securityContext().and()
  .requestCache().and()
  .anonymous().and()
  .servletApi().and()
  .apply(new DefaultLoginPageConfigurer<>()).and()
  .logout();
```

会将 `SecurityConfigurer` 的部分子类放到 `AbstractConfiguredSecurityBuilder` 的 `configurers` 中。例如 `anonymous` 就将 `AnonymousConfigurer` 放到了 `configurers` 中。然后

走到 `AbstractConfiguredSecurityBuilder` 的 `configure` 方法，简单来说就是调用所有 `configurers` 的 `configure` 的方法配置当前的 Builder ，还是拿 `AnonymousConfigurer` 为例，其就是在调用 `AnonymousAuthenticationFilter` 的 `afterPropertiesSet` 之后将其加入到 `HttpSecurity` 的 `filters` 中，然后封装成一个 `DefaultSecurityFilterChain` 过滤器链。然后将 `SecurityFilterChain` 封装到 `FilterChainProxy` 中。

在访问的时候首先会访问到 `DelegatingFilterProxy` 这个 Filter，在其 `doFilter` 方法中会调用 `initDelegate` 方法获取 ID 为 `springSecurityFilterChain`  类型为  `FilterChainProxy` 的 delegate 然后调用 `doFilter` 方法，然后调用其 `doFilterInternal` 方法，其中会获取上面所有的 Filters  `List<Filter> filters = getFilters(fwRequest);`，然后然后将这些 Filters 封装成一个 `VirtualFilterChain` 逐个调用, 之后就是调用`VirtualFilterChain` 的 `doFilter`方法了。



## Spring Security 的使用

在 pom 文件中引入相关的配置后就可以直接启动项目：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

默认会有一个密码打印在控制台中，测试的时候也可以直接在代码中定义一个用户：

```java
@Bean
@Override
public UserDetailsService userDetailsService() {
  UserDetails userDetails = User.withDefaultPasswordEncoder().username("xinyue")
    .password("xinyue").roles("USERS").build();
  return new InMemoryUserDetailsManager(userDetails);
}
```

然后再次启动就可以使用该用户名和密码登录了。如果要自定义登录页面可以使用在资源文件中添加出来相应的 page，并设置相应的 controller 简单的示例如下：

```java
// Controller
@RequestMapping("/ilogin")
public String ilogin() {
  return "ilogin";
}
```

```java
// Security 相关配置
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().authenticated()
                .and().formLogin(form ->
                	form.loginPage("/ilogin").permitAll()) 
                .csrf().disable(); // 禁用 CSRF
    }

    @Override
    public void configure(WebSecurity web) {
        web.ignoring().antMatchers("/static/**");
    }
}
```

上面第一个方法自定义了登录页面的地址为 `/ilogin` 页面，默认情况处理登录请求的地址也是 `/iloign` 只是 Method 是 POST，当然也可以通过相关的属性配置，例如 `loginProcessingUrl` 处理登录请求的 URL，`successForwardUrl` 登录成功之后跳转的地址，`defaultSuccessUrl`  也登录成功之后跳转的地址，只是其有一个默认为 false 的参数 `alwaysUse` 用于控制默认会跳转到登录前访问的页面 URL， `logoutSuccessUrl` 登出跳转地址，`failureForwardUrl` 登录失败跳转的地址。

在第二个方法表示过滤掉 `/static` 下所有的请求，用于处理静态资源等。

### CSRF

需要提示的一点是如果不想使用 `csrf` 一定要关闭 `csrf` 否则会被 Spring Security 认为是非法请求强制跳转到 `/error` 页面，如果要使用 `csrf` 需要在登录 form 中添加这样的隐藏字段，这里使用了 Thymeleaf 引擎，获取当前请求已经生成了一个 `token` 。

```java
<input type="hidden" name="_csrf" th:value="${_csrf.token}"/>
```

生成这个 Token 也是在每个终端当前 session 中第一次访问的时候，在 `CsrfFilter` 中的 `doFilterInternal` 方法中生成的。该方法会首先会从 `tokenRepository` 中去查找有没有已经生成过 token，如果已经生成则直接使用，如果没有生成过则调用 `this.tokenRepository.generateToken(request)` 为当前请求的生成一个 token，即生成一个 `DefaultCsrfToken` 对象，主要使用请求的 Header Name (`"X-CSRF-TOKEN"`)，Parameter Name （`_csrf`） 和随机数生成。

```java
// CsrfFilter
CsrfToken csrfToken = this.tokenRepository.loadToken(request);
final boolean missingToken = csrfToken == null;
if (missingToken) {
   csrfToken = this.tokenRepository.generateToken(request);
   this.tokenRepository.saveToken(csrfToken, request, response);
}
// HttpSessionCsrfTokenRepository
public CsrfToken generateToken(HttpServletRequest request) {
  return new DefaultCsrfToken(this.headerName, this.parameterName,
                              createNewToken());
}
private String createNewToken() {
  return UUID.randomUUID().toString();
}
```

因为默认使用的 `LazyCsrfTokenRepository` ，在生成完 session 紧随其后的 `saveToken` 方法中并不会真正的去保存 token 到 session 中，而是在 `getToken` 方法通过调用  `saveTokenIfNecessary` 方法最终调用 `HttpSessionCsrfTokenRepository` 的 `saveToken` 方法存到 session 中，因为如果没有使用 csrf 确实没有必要将这个 Token 放入的 Session 中。

另外就是在 `CsrfFilter` 中有一个 `DEFAULT_CSRF_MATCHER` 是 `DefaultRequiresCsrfMatcher` 其默认仅拦截除 `"GET", "HEAD", "TRACE", "OPTIONS"` 这些 Method 之外的请求。

```java
if (!this.requireCsrfProtectionMatcher.matches(request)) {
  filterChain.doFilter(request, response);
  return;
}
```

在非以上的请求中会从请求 Header 中去获取 csrf token 的值，和 session 中的比较确保不是其他页面的非法请求。

```java
String actualToken = request.getHeader(csrfToken.getHeaderName());
if (actualToken == null) {
  actualToken = request.getParameter(csrfToken.getParameterName());
}
if (!csrfToken.getToken().equals(actualToken)) {
  // token 不匹配的时候使用 accessDeniedHandler 跳转到
  if (missingToken) {
    this.accessDeniedHandler.handle(request, response,
                                    new MissingCsrfTokenException(actualToken));
  } else {
    this.accessDeniedHandler.handle(request, response,
                                    new InvalidCsrfTokenException(csrfToken, actualToken));
  }
  return;
}
```

## Spring Security 认证流程

默认的主要认证逻辑在 `UsernamePasswordAuthenticationFilter` 中完成的，可以看一下该类：

```java
public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
private boolean postOnly = true;

public UsernamePasswordAuthenticationFilter() {
  super(new AntPathRequestMatcher("/login", "POST"));
}
```

其定义了默认的用户名和密码参数，登录页面地址等信息。当然这些信息都可以通过上面的 `config` 方法修改，其实也就是通过 `FormLoginConfigurer` 进行修改的。其 `doFilter`  方法默认调用其父类的`AbstractAuthenticationProcessingFilter` 的方法，在其中完成了认证的操作。方法大概如下：

```java
// AbstractAuthenticationProcessingFilter
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
  throws IOException, ServletException {
  HttpServletRequest request = (HttpServletRequest) req;
  HttpServletResponse response = (HttpServletResponse) res;
  // 首先判断是否需要认证，如果不需要认证则直接进行下一步
  if (!requiresAuthentication(request, response)) {
    chain.doFilter(request, response);
    return;
  }
  Authentication authResult;
  try {
    // 如果需要认证则调用 attemptAuthentication 方法尝试认证
    authResult = attemptAuthentication(request, response);
    if (authResult == null) {
      return;
    }
    sessionStrategy.onAuthentication(authResult, request, response);
  } catch (InternalAuthenticationServiceException failed) {
    unsuccessfulAuthentication(request, response, failed);
    return;
  } catch (AuthenticationException failed) {
    unsuccessfulAuthentication(request, response, failed);
    return;
  }
  
  if (continueChainBeforeSuccessfulAuthentication) {
    chain.doFilter(request, response);
  }
  // 登出成功后续操作
  successfulAuthentication(request, response, chain, authResult);
}

protected void successfulAuthentication(HttpServletRequest request,
                                        HttpServletResponse response, FilterChain chain, Authentication authResult)
  throws IOException, ServletException {
	// 将认证信息放到 SecurityContextHolder 中
  SecurityContextHolder.getContext().setAuthentication(authResult);
	// 记住我设置
  rememberMeServices.loginSuccess(request, response, authResult);

  // 发布成功登陆事件
  if (this.eventPublisher != null) {
    eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
      authResult, this.getClass()));
  }
	
  successHandler.onAuthenticationSuccess(request, response, authResult);
}
```

下面看一下 `UsernamePasswordAuthenticationFilter` 的尝试认证的方法：

```java
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
  if (postOnly && !request.getMethod().equals("POST")) {
    throw new AuthenticationServiceException(
      "Authentication method not supported: " + request.getMethod());
  }
  // 获取请求中的用户名和密码
  String username = obtainUsername(request);
  String password = obtainPassword(request);
  if (username == null) {
    username = "";
  }
  if (password == null) {
    password = "";
  }
  username = username.trim();
  // 将用户名和密码封装成一个 UsernamePasswordAuthenticationToken 去认证
  UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
    username, password);
  // Allow subclasses to set the "details" property
  setDetails(request, authRequest);
  return this.getAuthenticationManager().authenticate(authRequest);
}
```

使用 `AuthenticationManager` 的 `authenticate` 进行认证。

```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
  Class<? extends Authentication> toTest = authentication.getClass();
  AuthenticationException lastException = null;
  AuthenticationException parentException = null;
  Authentication result = null;
  Authentication parentResult = null;

  // 获取所有 AuthenticationProvider 逐个判断是否 Support 当前的认证
  for (AuthenticationProvider provider : getProviders()) {
    if (!provider.supports(toTest)) {
      continue;
    }

    try {
      // 如果支持则使用 AuthenticationProvider 进行认证
      result = provider.authenticate(authentication);
      if (result != null) {
        copyDetails(authentication, result);
        break;
      }
    } catch (AccountStatusException | InternalAuthenticationServiceException e) {
      prepareException(e, authentication);
      throw e;
    } catch (AuthenticationException e) {
      lastException = e;
    }
  }

  if (result == null && parent != null) {
    // 如果当前 Provider 没有认证成功则使用其 parent 进行认证
    try {
      result = parentResult = parent.authenticate(authentication);
    } catch (ProviderNotFoundException e) { } catch (AuthenticationException e) {
      lastException = parentException = e;
    }
  }

  if (result != null) {
    if (eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
      ((CredentialsContainer) result).eraseCredentials(); // 清除认证信息
    }
    if (parentResult == null) {
      eventPublisher.publishAuthenticationSuccess(result); // 发布认证成功的事件
    }
    return result;
  }

  // 如果没有认证成功则抛出异常
  if (lastException == null) {
    lastException = new ProviderNotFoundException(messages.getMessage(
      "ProviderManager.providerNotFound",
      new Object[] { toTest.getName() },
      "No AuthenticationProvider found for {0}"));
  }
  if (parentException == null) {
    prepareException(lastException, authentication);
  }

  throw lastException;
}
```

 默认情况下会使用 `DaoAuthenticationProvider` 进行认证，首先使用 `AbstractUserDetailsAuthenticationProvider` 的 `authenticate` 主要方法如下：

```java
// AbstractUserDetailsAuthenticationProvider
public Authentication authenticate(Authentication authentication)
  throws AuthenticationException {
  // Determine username
  String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
    : authentication.getName();

  boolean cacheWasUsed = true;
  // 先尝试从 Cache 中获取用户
  UserDetails user = this.userCache.getUserFromCache(username);
  if (user == null) {
    cacheWasUsed = false;
    try {
      // 如果 Cache 中没有则根据用户名查找用户
      user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
    } catch (UsernameNotFoundException notFound) {
      // 没有找到用户
      if (hideUserNotFoundExceptions) {
        throw new BadCredentialsException(messages.getMessage(
          "AbstractUserDetailsAuthenticationProvider.badCredentials",
          "Bad credentials"));
      } else {
        throw notFound;
      }
    }
  }

  try {
    // 先检查账户是否过期，禁用，被锁等状态
    preAuthenticationChecks.check(user);
    // 后续认证，密码匹配等
    additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
  } catch (AuthenticationException exception) {
    if (cacheWasUsed) {
      // There was a problem, so try again after checking
      // we're using latest data (i.e. not from the cache)
      cacheWasUsed = false;
      user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
      preAuthenticationChecks.check(user);
      additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
    } else {
      throw exception;
    }
  }

  postAuthenticationChecks.check(user);
  if (!cacheWasUsed) {
    // 将用户放到 Cache 中
    this.userCache.putUserInCache(user);
  }
  Object principalToReturn = user;
  if (forcePrincipalAsString) {
    principalToReturn = user.getUsername();
  }
  // 生成成功登录的凭证
  return createSuccessAuthentication(principalToReturn, authentication, user);
}

protected Authentication createSuccessAuthentication(Object principal,
                                                     Authentication authentication, UserDetails user) {
  // Ensure we return the original credentials the user supplied,
  // so subsequent attempts are successful even with encoded passwords.
  // Also ensure we return the original getDetails(), so that future
  // authentication events after cache expiry contain the details
  UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
    principal, authentication.getCredentials(),
    authoritiesMapper.mapAuthorities(user.getAuthorities()));
  result.setDetails(authentication.getDetails());
  return result;
}
```

```java
// DaoAuthenticationProvider
protected final UserDetails retrieveUser(String username,
                                         UsernamePasswordAuthenticationToken authentication)
  throws AuthenticationException {
  prepareTimingAttackProtection();
  try {
    // 通过 UserDetailsService 根据用户名去查询 User信息
    // 我们在配置中定义了一个 UserDetailsService Bean 并将一个 User 传给了他
    // 所以这里能拿到配置用户的信息。
    UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
    if (loadedUser == null) {
      throw new InternalAuthenticationServiceException("UserDetailsService returned null, which is an interface contract violation");
    }
    return loadedUser;
  } catch (UsernameNotFoundException ex) {
    mitigateAgainstTimingAttack(authentication);
    throw ex;
  } catch (InternalAuthenticationServiceException ex) {
    throw ex;
  } catch (Exception ex) {
    throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
  }
}

protected void additionalAuthenticationChecks(UserDetails userDetails,
                                              UsernamePasswordAuthenticationToken authentication)
  throws AuthenticationException {
  // 从 authentication 获取认证信息
  if (authentication.getCredentials() == null) {
    logger.debug("Authentication failed: no credentials provided");
    throw new BadCredentialsException(messages.getMessage(
      "AbstractUserDetailsAuthenticationProvider.badCredentials",
      "Bad credentials"));
  }
  // 获取登录密码
  String presentedPassword = authentication.getCredentials().toString();
	// 使用 passwordEncoder 系统中的密码和请求提交的密码是否一致
  if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
    logger.debug("Authentication failed: password does not match stored value");
    throw new BadCredentialsException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
  }
}
```

以上涉及到类关系图如下：

![image-20200721220009230](https://i.loli.net/2020/07/21/PJGhbOl1iZudVFM.png)



###  从数据库中读取数据认证

通过上面的分析知道，如果要实现从数据库中获取认证内容。只要提供一个 `UserDetailService` 的实现即可，简单如下：

```java
// User 类
public class SysUser implements UserDetails {
    private int id;
    private String password;
    private String username;
```

```java
// Service 方法
@Service
public class SysUserService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        SysUser user = new SysUser(); // retrieve from db
        return user;
    }
}
```

显然如果明文存储密码是不安全的，所以创建用户的时候需要对用户的密码加密后存储。Spring Security 提供了一个 `PasswordEncoder` 并提供了很多的实现，这里介绍一下 `BCryptPasswordEncoder` 这个加密类。其提供了一下默认的构造方法，例如空，加密强度，加密版本，加密随机算法等。例如下面就是向容器中添加一个加密强度为 12 的 BCryptPasswordEncoder。

```java
@Bean
public PasswordEncoder passwordEncoder() {
  return new BCryptPasswordEncoder(12);
}
```

因为使用的是随机数作为盐，所以同一个字符串每次 `encode` 出来加密结果并不一致，其提供了 `matches` 方法，可以判断两个加密过的字符串的原本字符串是否相同。

