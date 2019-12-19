## Cookie 和 Session

### Cookie

#### Cookie 的作用

Cookie的作用像是去超市购物的时候办理的购物卡，里面村上了你的个人信息，下次来到超时后的时候，超市可以识别你的会员卡，直接购物。

由于 HTTP 是一种无状态协议，当用户的一次访问请求结束后，后端服务器无法知道下一次来访问的是否还是上次访问的用户。Cookie 的作用就是用来识别同一个用户，由于同一个客户端发出的请求都会带有第一次访问服务端设置的信息，这样服务端可以根据 Cookie 值来划分用户。

#### Cookie 的属性项

Cookie 分为两个版本：Version 0 和 Version 1，它们设置响应头的标识分别是 “Set-Cookie” 和 “Set-Cookie2”，这两个版本的属性值也有所不同，具体如下：

##### Version 0

属性列表如下：

1. NAME=VALUE： 键值对，可以设置要保存的 Key/Value 信息
2. Expires： 过期时间，设置某个时间点之后该 Cookie 过期
3. Domain：生成该 Cookie 的域名
4. Path：该 Cookie 是再当前哪个路径下生成的
5. Secure：如果设置了该属性，那么只会再 SSH 连接的时候才会回传该 Cookie

##### Version 1

除了包含 Version 0 的中除了Expires属性外的其他所有属性，还包括以下属性：

1. Version
2. Comment：注释项，说明该 Cookie 的用途
3. CommentURL：服务器为此 Cookie 提供的 URI 注释
4. Discard：是否再会话结束后丢弃该 Cookie 项，默认为 fasle
5. Max-Age：最大失效时间，这里设置的是多少秒后失效
6. Port：该 Cookie 再什么端口下可以回传给服务器，多个端口的时候使用逗号隔开

#### Cookie 的限制

Cookie 是 HTTP 头中一个字段，最终存储在浏览器中。首先浏览器对 Cookie 的大小和数量有限制，其次如果 Cookie 过大则会占用很多的宽带。另外由于 Cookie 存储在客户端浏览器中，所以这些 Cookie 可以被访问到，甚至被添加或者修改，使得 Cookie 不安全。



### Session

#### 为什么需要Session

Session 的出现是为了解决多次来回传输 Cookie 占用很多数据量的问题。同一个客户端每次和服务端交互时，不需要回传所有 Cookie，只需要回传一个客户端再第一次访问服务器时生成给 ID，这个 ID 时唯一的且每个客户端都有一个ID。通常时一个 NAME 为 JSESSIONID 的一个 Cookie。

#### Session 的工作模式

有以下三种方式让 Session 正常工作。

##### 基于 URL Path Parameter

浏览器不支持 Cookie 功能时，会将用户的 SessionCookieName 重写到请求的 URL 参数中，传递格式如 /path/Servlet;name=value;name2=value2?Nam3=value3 其中 "Servlet;" 后面的 K-V 就是要传递的 Path Parmaters，服务器会从 Path Patameters 中拿到用户配置的 SessionCookieName，默认的 SessionCookieNae 是 “JESSIONID”。拿到 Session ID 后会设置到 request.setRequestedSessionId 中。

##### 基于 Cookie

浏览器支持 Cookie 功能的时候会解析 Cookie 中的 SessionID。

##### 基于 SSL

默认不支持该模式，只有在 connector.getSAttribute("SSLEnabled")为 True 的时候才支持，会根据 javax.servlet.request.ssl_session 属性设置 SesssionID。

#### Session 的工作方式

有了 SessionID 服务端就可以创建 HttpSession 对象，第一次触发通过 request.getSessionI() 方法，如果当前的 SessionID 没有对应的 HttpSession 对象，那么就创建一个车新的，并加入到 session 容器中由 session 容器统一管理所有 session 的生命周期。另外只要这个 HttpSession 对象存在，就可以根据 Session ID 获取这个对象，也就可以做到状态的保持。

#### 分布式 Session 存储

对于大型网站而言单服务器肯定是无法满足需要的，所以需要一个分布式 Session 处理框架，将 Session 存储在一个分布式缓存中，可以随时写入和读取。

另外一个很重要的问题是如何处理跨域名来共享 Cookie 的问题，下图是一种解决方案。

![跨域名同步 Session](https://i.loli.net/2019/04/05/5ca76e15259ee.png)



另外对于多终端共享 Session 的情况，必须要有统一的会话架构。在服务端统一 Session 后，不管是访问哪个服务端都能做到登录状态的统一，可以使用统一的 Session ID 去服务端验证登录状态。多终端的登陆情况可以使用扫码登陆，如下图所示：

![多终端扫描登陆示意图](https://i.loli.net/2019/04/05/5ca76fe0e8af6.png)



参考文章：

[深入分析Java Web技术内幕（修订版）](https://book.douban.com/subject/25953851/)





