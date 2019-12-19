## Nginx proxy pass, root alias

[TOC]

### proxy_pass
proxy_pass 一般有四种设置方式, 例如访问 index.html 页面：

1 代理 URL： http://127.0.0.1/index.html
```nginx
location /proxy/ {
    proxy_pass http://127.0.0.1/;
}
```
2 代理 URL： http://127.0.0.1/proxy/index.html
```nginx
location /proxy/ {
    proxy_pass http://127.0.0.1;
}
```
3 代理 URL：http://127.0.0.1/demo/index.html
```nginx
location /proxy/ {
    proxy_pass http://127.0.0.1/demo/;
}
```
4 代理 URL：http://127.0.0.1/proxy/demo/index.html
```nginx
location /proxy/ {
    proxy_pass http://127.0.0.1/demo/;
}
```

参考：[nginx 之 proxy_pass详解](https://blog.csdn.net/zhongzh86/article/details/70173174)

### root 与 alias
全局配置如下：
```nginx
server {
    listen 80 default_server;
    server_name localhost;
    root /var/www/html;
    index index.html;
}

```
html 下有一个 blog 文件夹，demo 中有一个 demo.html

在为配置任何规则的情况下访问 `localhost/demo/demo.html` 可以正常访问网页。
默认会使用全局的 root 路径拼接上 url 地址获取，拼接之后是 `/var/www/html/demo/demo.html`

如果配置了如下的 rule :
```nginx
# config 1
location ^~ /demo {
    root /var/www/html;
}

# config 2
location ^~ /demo {
    root /var/www/html/;
}

# config 3
location ^~ /demo/ {
    root /var/www/html;
}

# config 4
location ^~ /demo/ {
    root /var/www/html/;
}

```
以上几种配置方式和无配置一致。对于 config 2/4 来说拼接后是 `/var/www/html//demo/demo.html` 多个 / 会认作 1 个。

如果配置了如下的 rule:
```nginx
location ^~ demo/ {
    root /var/www/html;
}
```
这样的规则是无法被匹配到，会走全局的 root 规则。

所以对于对于匹配的地址，会将 url 中路径拼接上 rule 中配置的 路径就得到了真实页面的地址。如果找不到 rule 则使用默认的 root。

对于 alias 来说是使用 alias 后的值替换匹配的 URL 的路径得到真实页面的地址，例如访问 `localhost/demo/demo.html`
例如以下几条rule, 规则如下：
```nginx
#config 1
location ^~ /demo {
    alias /var/www/html;
}
# 查询地址：/var/www/html/demo.html

#config 2
location ^~ /demo {
    alias /var/www/html/;
}
# 查询地址：/var/www/html//demo.html

#config 3
location ^~ /demo/ {
    alias /var/www/html;
}
# 查询地址：/var/www/htmldemo.html

#config 4
location ^~ /demo/ {
    alias /var/www/html/;
}
# 查询地址：/var/www/html/demo.html
```