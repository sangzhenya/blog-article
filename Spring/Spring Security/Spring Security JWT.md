---
title: "Spring Security JWT"
tags: ["Spring", "Spring Security"]
categories: ["Spring"]
date: "2020-05-17T09:00:00+08:00"
toc: true
---

[toc]

## JWT

JWT (JSON Web Token) 是出色的分布式身份校验方案，可以生成 Token，也可以解析校验 Token。

JWT 生成的 Token 由三部分组成：

1. 头部

   主要设置一些规范信息，签名部分的格式编码就在头部中声明

2. 载荷

   Token 中存有有效信息的部分，用户名，用户角色，过期时间等。

3. 签名

   将头部和载荷分别采用 base64加密后，用 “.” 相连，再加入盐，最后使用头部声明的编码类型进行编码，就可以得到签名了。

如果避免 JWT 生成的 Token 被伪造主要看签名的部分，其中头部和载荷是 base64 编码不具有安全性，所以重点就是加密时候使用的盐了。如果加解密使用的盐是一样的则很危险可能被用来伪造 Token，所以需要使用非对称加密，即生成 Token 的盐与校验 Token 的盐不一样。

##  RSA 非对称加密

基本原理是：同时生成两把秘钥：私钥和公钥。私钥私密保存，公钥下发到客户端。公钥加密的时候只有私钥才能解密。私钥加密的时候公钥或者私钥可以解密。这个虽然安全，但是算法比较耗时。

## 基于 JWT 和 RSA 的登录实现

首先看一下 pom 文件：

```xml
<!-- 部分 dependency 如下 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
  <exclusions>
    <exclusion>
      <groupId>org.junit.vintage</groupId>
      <artifactId>junit-vintage-engine</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt</artifactId>
  <version>0.9.1</version>
</dependency>
<dependency>
  <groupId>commons-io</groupId>
  <artifactId>commons-io</artifactId>
  <version>2.7</version>
</dependency>
```



### 添加基于 RSA 生成 Key 的方法

生成 Private Key 和 Public Key 的文件的代码如下：

```java
private static final int DEFAULT_KEY_SIZE = 2048;
public static void generateKey(String publicKeyFileName, String privateKeyFileName, String secret, int keySize) throws NoSuchAlgorithmException, IOException {
   KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
   SecureRandom secureRandom = new SecureRandom(secret.getBytes());
   keyPairGenerator.initialize(Math.max(keySize, DEFAULT_KEY_SIZE), secureRandom);
   KeyPair keyPair = keyPairGenerator.genKeyPair();
   byte[] publicKeys = keyPair.getPublic().getEncoded();
   FileUtils.writeByteArrayToFile(new File(publicKeyFileName), Base64.getEncoder().encode(publicKeys));
   byte[] privateKeys = keyPair.getPrivate().getEncoded();
   FileUtils.writeByteArrayToFile(new File(privateKeyFileName), Base64.getEncoder().encode(privateKeys));
 }
```

然后通过一个测试方法生成 private key 和 public key

```java
private static final String PRIVATE_KEY_FILE = "/Users/xinyue/temp/rsa/id_rsa";
private static final String PUBLIC_KEY_FILE = "/Users/xinyue/temp/rsa/id_rsa.pub";

@Test
public void generateKeys() {
    try {
        RsaUtils.generateKey(PUBLIC_KEY_FILE, PRIVATE_KEY_FILE, "XINYUE", 2048);
    } catch (NoSuchAlgorithmException | IOException e) {
        e.printStackTrace();
    }
}
```

### 添加基本的 Domain 类和一些 Util 方法

权限类

```java
public class SysRole implements GrantedAuthority {
    private String authority;
```

用户类

```java
public class SysUser implements UserDetails {
    private int id;
    private String password;
    private String username;
    private boolean accountNonExpired = true;
    private boolean accountNonLocked = true;
    private boolean credentialsNonExpired = true;
    private boolean enabled = true;
    private List<SysRole> authorities;
```

为 JWT 封装的一个类

```java
public class Payload<T> {
    private String id;
    private T userInfo;
    private LocalDateTime expiration;
```

JSON Util 用户转换 Bean2Json 和 Json2Bean

```java
public class JsonUtil {
    public static final ObjectMapper mapper = new ObjectMapper();
    private static final Logger logger = LoggerFactory.getLogger(JsonUtil.class);

    public static String toString(Object obj) {
        if (obj == null) {
            return null;
        }
        if (obj.getClass() == String.class) {
            return (String) obj;
        }
        try {
            return mapper.writeValueAsString(obj);
        } catch (Exception e) {
            logger.error("Error:", e);
        }
        return null;
    }

    public static <T> T toBean(String json, Class<T> tClass) {
        try {
            return mapper.readValue(json, tClass);
        } catch (Exception e) {
            logger.error("Error:", e);
        }
        return null;
    }
}
```

### 添加 RsaKeyProperties 用于获取 Public Key 和 Private Key

配置类

```java
@ConfigurationProperties(prefix = "rsa.key")
public class RsaKeyProperties {
    private String priKeyFile;
    private String pubKeyFile;

    private PublicKey publicKey;
    private PrivateKey privateKey;

    @PostConstruct
    public void createKeys() {
        try {
            privateKey = RsaUtils.getPrivateKey(priKeyFile);
            publicKey = RsaUtils.getPublishKey(pubKeyFile);
        } catch (Exception e) {
            LoggerFactory.getLogger(this.getClass()).error("Error:", e);
        }
    }
```

```java
public class RsaUtils {
  	// 获取 Public Key
    public static PublicKey getPublishKey(String fileName) throws IOException, InvalidKeySpecException, NoSuchAlgorithmException {
        return getPublishKey(FileUtils.readFileToString(new File(fileName)).getBytes());
    }
    private static PublicKey getPublishKey(byte[] bytes) throws NoSuchAlgorithmException, InvalidKeySpecException {
        bytes = Base64.getDecoder().decode(bytes);
        X509EncodedKeySpec spec = new X509EncodedKeySpec(bytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return keyFactory.generatePublic(spec);
    }
  
  	// 获取 Private Key
    public static PrivateKey getPrivateKey(String fileName) throws IOException, InvalidKeySpecException, NoSuchAlgorithmException {
        return getPrivateKey(FileUtils.readFileToString(new File(fileName)).getBytes());
    }
    private static PrivateKey getPrivateKey(byte[] bytes) throws NoSuchAlgorithmException, InvalidKeySpecException {
        bytes = Base64.getDecoder().decode(bytes);
        PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(bytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return keyFactory.generatePrivate(spec);
    }
```

Main 类中引入配置

```java
@EnableConfigurationProperties(RsaKeyProperties.class)
@SpringBootApplication
public class SecurityApplication {
```

Yaml 中配置

```yaml
rsa:
  key:
    pri-key-file: /Users/xinyue/temp/rsa/id_rsa
    pub-key-file: /Users/xinyue/temp/rsa/id_rsa.pub
```

### 添加两个 Filter 分别用于登录和认证

#### 登录 Filter

首先看一下登录的 Filter 直接继承  `UsernamePasswordAuthenticationFilter`

```java
public class JwtLoginFilter extends UsernamePasswordAuthenticationFilter {
    private static final Logger logger = LoggerFactory.getLogger(JwtLoginFilter.class);
    private final AuthenticationManager authenticationManager;
    private final RsaKeyProperties rsaKeyProperties;

    public JwtLoginFilter(AuthenticationManager authenticationManager, RsaKeyProperties rsaKeyProperties) {
        this.authenticationManager = authenticationManager;
        this.rsaKeyProperties = rsaKeyProperties;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
      	SysUser sysUser = new SysUser();
        try {
            sysUser = new ObjectMapper().readValue(request.getInputStream(), SysUser.class);
        } catch (Exception e) {
            logger.error("Error:", e);
        }
        // 从 Header 中获取 User 并调用默认的认证方法
        return this.authenticationManager.authenticate(new UsernamePasswordAuthenticationToken(sysUser.getUsername(), sysUser.getPassword()));
    }

    @Override
    public void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {
				// 登录成功后将 Token 放到 Header 中返回给客户端
      	SysUser sysUser = new SysUser();
        sysUser.setUsername(authResult.getName());
        sysUser.setAuthorities((List<SysRole>) authResult.getAuthorities());
        String token = JwtUtil.generateTokenExpireInSeconds(sysUser, rsaKeyProperties.getPrivateKey(), 2 * 60);
        response.setHeader(JwtConstant.HEADER_STRING, JwtConstant.TOKEN_PREFIX + token);
        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        Map<String, Object> resMap = new HashMap<>();
        resMap.put("status", 200);
        resMap.put("message", "Success");
        response.getOutputStream().println(JsonUtil.toString(resMap));
    }

    @Override
    public void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response, AuthenticationException failed) throws IOException, ServletException {
      	// 登录失败返回错误给客户端
        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        Map<String, Object> resMap = new HashMap<>();
        resMap.put("status", 403);
        resMap.put("message", "Unauthorized");
        response.getOutputStream().println(JsonUtil.toString(resMap));
    }
}
```

```java
// JwtUtil
private static final String JWT_PAYLOAD_USER_KEY = "user";
// 根据 Private Key 生成 Token
public static String generateTokenExpireInSeconds(Object userInfo, PrivateKey privateKey, int expire) {
  return Jwts.builder()
    .claim(JWT_PAYLOAD_USER_KEY, JsonUtil.toString(userInfo))
    .setAudience(createJTI())
    .setExpiration(Date.from(LocalDateTime.now().plusSeconds(expire).atZone(ZoneId.systemDefault()).toInstant()))
    .signWith(SignatureAlgorithm.RS256, privateKey)
    .compact();
}
private static String createJTI() {
  return new String(Base64.getEncoder().encode(UUID.randomUUID().toString().getBytes()));
}
```

如果要自定义认证的 URL 可以将 `JwtLoginFilter` 继承 `AbstractAuthenticationProcessingFilter`， 然后通过 `super(new AntPathRequestMatcher("/login", "POST"));`  的方式自定义登录地址。

#### 认证 Filter

然后添加用于认证的 Filter 继承 `BasicAuthenticationFilter`

```java
public class JwtAuthenticationFilter extends BasicAuthenticationFilter {
    private final RsaKeyProperties rsaKeyProperties;
    public JwtAuthenticationFilter(AuthenticationManager authenticationManager, RsaKeyProperties rsaKeyProperties) {
        super(authenticationManager);
        this.rsaKeyProperties = rsaKeyProperties;
    }

    @Override
    public void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
	      // 从请求 header 中获取 Token 信息
        String header = request.getHeader(JwtConstant.HEADER_STRING);
        if (!StringUtils.isEmpty(header) && header.startsWith(JwtConstant.TOKEN_PREFIX)) {
          	// 解析 Token 信息
            Payload<SysUser> payload = JwtUtil.getInfoFromToken(header.replace(JwtConstant.TOKEN_PREFIX, ""), rsaKeyProperties.getPublicKey(), SysUser.class);
            if (payload.getUserInfo() != null) {
                SysUser userInfo = payload.getUserInfo();
	              // 将 Token 信息放到 Context 中
                SecurityContextHolder.getContext().setAuthentication(new UsernamePasswordAuthenticationToken(userInfo.getUsername(), null, userInfo.getAuthorities()));
            }
        }
        chain.doFilter(request, response);
    }
}
```

```java
// JwtUtil
private static final String JWT_PAYLOAD_USER_KEY = "user";
// 使用 Public Key 解析字符串获取 Token 信息
public static <T> Payload<T> getInfoFromToken(String token, PublicKey publicKey, Class<T> userType) {
  Jws<Claims> claimsJws = parserToken(token, publicKey);
  Claims body = claimsJws.getBody();
  Payload<T> claims = new Payload<>();
  claims.setId(body.getId());
  claims.setUserInfo(JsonUtil.toBean(body.get(JWT_PAYLOAD_USER_KEY).toString(), userType));
  claims.setExpiration(LocalDateTime.ofInstant(body.getExpiration().toInstant(), ZoneId.systemDefault()));
  return claims;
}
private static Jws<Claims> parserToken(String token, PublicKey key) {
  return Jwts.parser().setSigningKey(key).parseClaimsJws(token);
}
```

### 配置 Security Config

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
  @Autowired
  private RsaKeyProperties properties;
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.csrf().disable()
      .authorizeRequests().anyRequest().authenticated().and()
      // 将两个 Filter 添加到过滤器链中
      .addFilter(new JwtLoginFilter(authenticationManager(), properties))
      .addFilter(new JwtAuthenticationFilter(authenticationManager(), properties));
  }
}
```

### 添加 UserDetailsService

```java
@Service
public class SysUserService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        SysUser sysUser = // 从数据库中获取 User
        return sysUser;
    }
}
```

## 登录测试

编写 http 方法, 使用 Intellij IDEA 执行

### 登录方法

```
POST http://localhost:8081/login
Accept: */*
Cache-Control: no-cache
Content-Type: application/json

{
  "username": "xinyue",
  "password": "xinyue"
}
```

结果如下：

```
POST http://localhost:8081/login

HTTP/1.1 401 
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJ1c2VyIjoie1wiaWRcIjowLFwicGFzc3dvcmRcIjpudWxsLFwidXNlcm5hbWVcIjpcInhpbnl1ZVwiLFwiYWNjb3VudE5vbkV4cGlyZWRcIjp0cnVlLFwiYWNjb3VudE5vbkxvY2tlZFwiOnRydWUsXCJjcmVkZW50aWFsc05vbkV4cGlyZWRcIjp0cnVlLFwiZW5hYmxlZFwiOnRydWUsXCJhdXRob3JpdGllc1wiOlt7XCJhdXRob3JpdHlcIjpcImRlbW9cIn0se1wiYXV0aG9yaXR5XCI6XCJhZG1pblwifV19IiwiYXVkIjoiTVRFMlptSXhZamd0WVRreE1pMDBNR1JpTFdKak4yTXRPVFF5TmpJME9HRXdNVEF6IiwiZXhwIjoxNTk1NzQ3OTMwfQ.SlS0SG6QPpLzL8hpNy5MGiBHH58TYN7EgZnsIFo4oUp_u72A8wDtiRxS1UdvjgFGfl7T0seOCLv1BcnoIae_6QrSsBd8sLdSuzFnQhXMKlL4VOvaSpqC9km0jqFBMIIwnKm1cLYF-wiTFHeCyi03gp3XRkFk6EC0e0Gn1LC6VatTbH6nXM2Ff66kOn9NQv5rX1IN3O2AUGa8JLcng0wAL1Hpize_S5u4_eriuqG3gJiI1XIJ7qVo86_tARC5ZEOqEDcb4BMN8YtPw4gpuBxfNMmj0FUSXS44Fxwv7JfOUueguHpdKeGurZifnut7HciW4fRklKsDyP0ANMUhXlK8ng
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Type: application/json
Content-Length: 36
Date: Sun, 26 Jul 2020 07:16:50 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "message": "Success",
  "status": 200
}


Response code: 401; Time: 3485ms; Content length: 36 bytes
```

### 使用 Authorization 验证

然后将 Authorization 放到后续的请求 Header

```
GET http://localhost:8081/maintain
Accept: */*
Cache-Control: no-cache
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJ1c2VyIjoie1wiaWRcIjowLFwicGFzc3dvcmRcIjpudWxsLFwidXNlcm5hbWVcIjpcInhpbnl1ZVwiLFwiYWNjb3VudE5vbkV4cGlyZWRcIjp0cnVlLFwiYWNjb3VudE5vbkxvY2tlZFwiOnRydWUsXCJjcmVkZW50aWFsc05vbkV4cGlyZWRcIjp0cnVlLFwiZW5hYmxlZFwiOnRydWUsXCJhdXRob3JpdGllc1wiOlt7XCJhdXRob3JpdHlcIjpcImRlbW9cIn0se1wiYXV0aG9yaXR5XCI6XCJhZG1pblwifV19IiwiYXVkIjoiTVRFMlptSXhZamd0WVRreE1pMDBNR1JpTFdKak4yTXRPVFF5TmpJME9HRXdNVEF6IiwiZXhwIjoxNTk1NzQ3OTMwfQ.SlS0SG6QPpLzL8hpNy5MGiBHH58TYN7EgZnsIFo4oUp_u72A8wDtiRxS1UdvjgFGfl7T0seOCLv1BcnoIae_6QrSsBd8sLdSuzFnQhXMKlL4VOvaSpqC9km0jqFBMIIwnKm1cLYF-wiTFHeCyi03gp3XRkFk6EC0e0Gn1LC6VatTbH6nXM2Ff66kOn9NQv5rX1IN3O2AUGa8JLcng0wAL1Hpize_S5u4_eriuqG3gJiI1XIJ7qVo86_tARC5ZEOqEDcb4BMN8YtPw4gpuBxfNMmj0FUSXS44Fxwv7JfOUueguHpdKeGurZifnut7HciW4fRklKsDyP0ANMUhXlK8ng
```

结果如下

```
GET http://localhost:8081/maintain

HTTP/1.1 200 
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Type: text/plain;charset=UTF-8
Content-Length: 8
Date: Sun, 26 Jul 2020 07:17:57 GMT
Keep-Alive: timeout=60
Connection: keep-alive

maintain

Response code: 200; Time: 56ms; Content length: 8 bytes
```

## OAuth 2

OAuth (Open Authorization) 其为用户提供了一个安全开放，又简易的标准。与以往的授权方式不同之处是其的授权不会使第三方初级到用户的账号信息，用户名和密码等。即第三方无需使用用户名与密码就可以申请该用户资源的授权，所以 OAuth 是安全的。

1. 授权码模式

   前提：A 服务客户端需要用到 B 服务资源服务中的资源。

   1. A 服务客户端将用户自动导航到 B 服务认证服务，这一步用户需要提供一个回调地址，以备 B 服务认证服务返回授权码使用。
   2. 用户点击授权按钮表示让 A 服务客户端使用 B 服务资源服务，这一步需要用户登录 B 服务，也就是说用户要事先具有 B 服务的使用权限。
   3. B 认证服务服务生成授权码，授权码通过第一步提供的回调地址，返回给 A 服务客户端，但是这个授权并非同行 B 服务资源服务器的通行凭证。
   4. A 服务认证服务携带上一步得到的授权向 B 服务认证服务发送请求，获取拖行凭证 Token。
   5. B 服务认证服务给 A 服务认证服务返回令牌 Token 和更新 Refresh Token。

   适用场景：授权码模式是 OAuth 2 中最安全最完善的一种模式，应用场景广泛，实现服务之间的调用，微信，QQ 等第三方登录也是采用这种方式。

2. 简化模式

   说明：简化模式中没有 A 服务器认证服务这一部分，全部由 A 服务器客户端与 B 服务交互，整个过程不再有授权码，Token 直接暴露在浏览器中。

   1. A 服务客户端将用户自动导航到 B 服务认证服务，这一步用户需要提供一个回调地址，以备 B 服务认证服务返回授权码使用。
   2. 用户点击授权按钮表示让 A 服务客户端使用 B 服务资源服务，这一步需要用户登录 B 服务，也就是说用户要事先具有 B 服务的使用权限。
   3. B 服务认证服务生成 Token，Token 通过第一步的回调地址返回给 A 服务客户端。

3. 密码模式

   1. 直接告诉 A 服务客户端自己 B 服务认证服务的用户名和密码
   2. A 服务客户端携带 B 服务认证服务的用户名和密码箱 B 服务认证服务发起请求获取 Token
   3. B 服务认证服务给 A 服务客户端 Token

4. 客户端模式

   1. A 服务直接向 B 服务索取 Token
   2. B 服务返回 Token 给 A 服务



## CSA SSO 

参考： [单点登录（SSO）看这一篇就够了](https://developer.aliyun.com/article/636281)

SSO (Single Sign On) 单点登录即在多个应用系统中，只需要登录一次，就可以访问其他相互信任的应用系统。

流程如下：

1. 用户访问app系统，app系统是需要登录的，但用户现在没有登录。
2. 跳转到CAS server，即SSO登录系统，**以后图中的CAS Server我们统一叫做SSO系统。** SSO系统也没有登录，弹出用户登录页。
3. 用户填写用户名、密码，SSO系统进行认证后，将登录状态写入SSO的session，浏览器（Browser）中写入SSO域下的Cookie。
4. SSO系统登录完成后会生成一个ST（Service Ticket），然后跳转到app系统，同时将ST作为参数传递给app系统。
5. app系统拿到ST后，从后台向SSO发送请求，验证ST是否有效。
6. 验证通过后，app系统将登录状态写入session并设置app域下的Cookie。

至此，跨域单点登录就完成了。以后我们再访问app系统时，app就是登录的。接下来，我们再看看访问app2系统时的流程。

1. 用户访问app2系统，app2系统没有登录，跳转到SSO。
2. 由于SSO已经登录了，不需要重新登录认证。
3. SSO生成ST，浏览器跳转到app2系统，并将ST作为参数传递给app2。
4. app2拿到ST，后台访问SSO，验证ST是否有效。
5. 验证成功后，app2将登录状态写入session，并在app2域下写入Cookie。

这样，app2系统不需要走登录流程，就已经是登录了。SSO，app和app2在不同的域，它们之间的session不共享也是没问题的。