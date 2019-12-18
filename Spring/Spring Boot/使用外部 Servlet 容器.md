## 使用外部容器

在创建项目的时候使用 war 的方式打包：

![项目创建](http://img.sangzhenya.com/Snipaste_2019-12-18_22-13-39.png)

其实主要的 pom 文件的配置如下：

```xml
<groupId>com.example</groupId>
<artifactId>webwar</artifactId>
<version>0.0.1-SNAPSHOT</version>
<!--打包方式设置为 war-->
<packaging>war</packaging>

<!--将内置的 Tomcat 设置为 provided -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-tomcat</artifactId>
  <scope>provided</scope>
</dependency>
```

在项目设置中设置 Web Source 的目录

![Web Source](http://img.sangzhenya.com/Snipaste_2019-12-18_22-19-06.png)

然后设置 Edit Configuration

![设置 Tomcat](http://img.sangzhenya.com/Snipaste_2019-12-18_22-21-18.png)

在 `src/main/webapp/`添加 `index.jsp` 文件，然后启动就可以在浏览器中看到成功的页面。

![成功页面](http://img.sangzhenya.com/Snipaste_2019-12-18_22-24-36.png)