---
title: "MyBaits 插件"
tags: ["MyBaits"]
categories: ["MyBaits"]
date: "2019-05-03T09:00:00+08:00"
---

MyBatis 在创建四个主要对象（`Executor`， `ParameterHandler`, `ResultSetHandler`, `StatementHandler`）的过程中都会使用到插件。插件可以利用动态代理机制一层层的包装目标对象从而实现在目标对象执行目标方法之前进行拦截。

一个简单的插件编写如下：

首先编写实现 `Interceptor` 的主类：

```java
// 这里指定拦截 StatementHandler 的 parameterize 方法且参数是 Statement
@Intercepts({@Signature(type = StatementHandler.class, method = "parameterize", args = Statement.class)})
public class MyInterceptor implements Interceptor {
    private Logger logger = LoggerFactory.getLogger(MyInterceptor.class);
    // 拦截目标方法的执行
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        logger.info("invocation method: " + invocation.getMethod().getName());
        Object target = invocation.getTarget();
        MetaObject metaObject = SystemMetaObject.forObject(target);
        Object value = metaObject.getValue("parameterHandler.parameterObject");
        if (value instanceof Integer) {
            metaObject.setValue("parameterHandler.parameterObject", (Integer)value + 1 );
        }
      	// 继续执行方法
        return invocation.proceed();
    }

    // 包装为目标对象创建的代理对象
    @Override
    public Object plugin(Object target) {
        logger.info("Use plugin");
	      // 包装对象
        return Plugin.wrap(target, this);
    }

    // 从配置中为当前插件设置的属性会传递到这边
    @Override
    public void setProperties(Properties properties) {
        logger.info("Set properties");
        logger.info(properties.toString());
    }
}

// MyBatis 提供的 wrap 方法
public static Object wrap(Object target, Interceptor interceptor) {
  // 简单来讲就是判断当前 target 的方法是否是自定义接口中需要的方法
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  Class<?> type = target.getClass();
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
      type.getClassLoader(),
      interfaces,
      new Plugin(target, interceptor, signatureMap));
  }
  return target;
}
```

然后再 MyBaits 的 config 中注册该方法：

```xml
<plugins>
  <plugin interceptor="com.xinyue.imybatis.interceptor.MyInterceptor">
    <!-- 传递给 interceptor 的属性 -->
    <property name="password" value="123456"/>
  </plugin>
</plugins>
```

然后正常运行就可以调用到这里自定的拦截方法了。

具体调用定义插件的地方是四个主要对象生成的时候都会调用的一个方法

```java
public Object pluginAll(Object target) {
  // 即获取出所有的插件逐个调用其 plugin 包装目标对象带代理对象
  for (Interceptor interceptor : interceptors) {
    target = interceptor.plugin(target);
  }
  return target;
}
```

如果定义多个插件则其包装图如下：

![多个插件包装图](http://img.programya.com/20191229100858.png)

即先执行后定义的，再执行先定义的，最后执行目标方法。