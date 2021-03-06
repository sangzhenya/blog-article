---
title: "MyBatis 运行原理"
tags: ["MyBaits"]
categories: ["MyBaits"]
date: "2019-05-11T09:00:00+08:00"
---

MyBaits 分层结构图

![MyBatis 分层结构图](http://img.programya.com/20191228151822.png)

MyBatis 过程整个过程有以下四步：

1. 创建 `SqlSessionFactory`
2. 获取 `SqlSession`
3. 获取代理对象
4. 执行方法

### 获取 `SqlSessionFactory` 

```java
public SqlSessionFactory getSqlSessionFactory() throws IOException {
  // 获取配置的文件流
  InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
  // 根据配置使用 SqlSessionFactoryBuilder 生成 SqlSessionFactory
  return new SqlSessionFactoryBuilder().build(inputStream);
}
```

```java
// SqlSessionFactoryBuilder
public SqlSessionFactory build(InputStream inputStream) {
  // 调用内部的 build 方法
  return build(inputStream, null, null);
}
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // 根据配置文件，环境信息和配置信息生成一个 XMLConfigBuilder
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    // 解析文件生成 Configuration 对象，然后使用 Configuration 放到 SqlSessionFactory 返回
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      inputStream.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
public SqlSessionFactory build(Configuration config) {
  return new DefaultSqlSessionFactory(config);
}
```

```java
// XMLConfigBuilder
public Configuration parse() {
  // 判断是否解析过，如果解析过则抛出异常
  if (parsed) {
    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
  }
  parsed = true;
  // 获取 MyBaits Config 的根节点 configuration，解析 config
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}
private void parseConfiguration(XNode root) {
  try {
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    // 如果配置了 vfsImpl，取出来放到 config 上
    loadCustomVfs(settings);
    // 设置自定义日志的实现 SLF4J | LOG4J | LOG4J2 | JDK_LOGGING | COMMONS_LOGGING | STDOUT_LOGGING | NO_LOGGING
    loadCustomLogImpl(settings);
    typeAliasesElement(root.evalNode("typeAliases"));
    pluginElement(root.evalNode("plugins"));
    // 解析对象工厂
    objectFactoryElement(root.evalNode("objectFactory"));
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    typeHandlerElement(root.evalNode("typeHandlers"));
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
// 处理 properties 配置
private void propertiesElement(XNode context) throws Exception {
  if (context != null) {
    // 获取所有的属性
    Properties defaults = context.getChildrenAsProperties();
    // 获取 resource 属性和 url 属性
    String resource = context.getStringAttribute("resource");
    String url = context.getStringAttribute("url");
    if (resource != null && url != null) {
      throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
    }
    // 如果 resource 不为空则获取所有的属性到 defaults 中，如果配置了 url 则从 url 中读取属性到 resource 中
    // 这里也可以看出，从 resource 中读取的属性会覆盖 properties 内部配置的属性
    if (resource != null) {
      defaults.putAll(Resources.getResourceAsProperties(resource));
    } else if (url != null) {
      defaults.putAll(Resources.getUrlAsProperties(url));
    }
    // 如果调用 build 方法的时候传入了参数，这边也会放入到属性上
    Properties vars = configuration.getVariables();
    if (vars != null) {
      defaults.putAll(vars);
    }
    // 将所有的属性放在 parser 和 configuration 上
    parser.setVariables(defaults);
    configuration.setVariables(defaults);
  }
}
// 处理 setting 配置
private Properties settingsAsProperties(XNode context) {
  if (context == null) {
    return new Properties();
  }
  // 获取所有的 setting 值
  Properties props = context.getChildrenAsProperties();
  // 确保所有的设置都是可用的，即 MyBatis 支持的，否则抛出异常
  MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
  for (Object key : props.keySet()) {
    if (!metaConfig.hasSetter(String.valueOf(key))) {
      throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
    }
  }
  return props;
}
// 设置别名信息
private void typeAliasesElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 如果配置的是 package
      if ("package".equals(child.getName())) {
        String typeAliasPackage = child.getStringAttribute("name");
        // 扫描包逐个注册 alias 和 type
        configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
      } else {
        // 如果是逐个注册 alias 和 type
        String alias = child.getStringAttribute("alias");
        String type = child.getStringAttribute("type");
        try {
          Class<?> clazz = Resources.classForName(type);
          if (alias == null) {
            typeAliasRegistry.registerAlias(clazz);
          } else {
            typeAliasRegistry.registerAlias(alias, clazz);
          }
        } catch (ClassNotFoundException e) {
          throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
        }
      }
    }
  }
}
// 添加 plugin 
private void pluginElement(XNode parent) throws Exception {
  if (parent != null) {
    // 遍历配置
    for (XNode child : parent.getChildren()) {
      // 获取配置的 interceptor
      String interceptor = child.getStringAttribute("interceptor");
      // 获取配置的属性
      Properties properties = child.getChildrenAsProperties();
      // 生成一个 Interceptor 并设置属性，然后放到 configuration 上
      Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
      interceptorInstance.setProperties(properties);
      configuration.addInterceptor(interceptorInstance);
    }
  }
}
// 从 setting 上读取配置信息逐个放到 config 上
private void settingsElement(Properties props) {
  configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
  configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
  configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
  configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
  configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
  configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
  configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
  configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
  configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
  configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
  configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
  configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
  configuration.setDefaultResultSetType(resolveResultSetType(props.getProperty("defaultResultSetType")));
  configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
  configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
  configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
  configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
  configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
  configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
  configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
  configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
  configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
  configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
  configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
  configuration.setLogPrefix(props.getProperty("logPrefix"));
  configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
}
// 解析 environments 元素
private void environmentsElement(XNode context) throws Exception {
  if (context != null) {
    if (environment == null) {
      environment = context.getStringAttribute("default");
    }
    // 遍历 environments
    for (XNode child : context.getChildren()) {
      // 获取每个 environment 的 id
      String id = child.getStringAttribute("id");
      // 判断是否是指定的 id，就是配置中配置的
      if (isSpecifiedEnvironment(id)) {
        // 获取 TransactionFactory
        TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
        // 根据 dataSource 的配置信息创建 DataSourceFactory，然后从 DataSourceFactory 中取出 DataSource
        // 数据库的配置信息在这边读取
        DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
        DataSource dataSource = dsFactory.getDataSource();
        // 生成 Environment.Builder 然后通过其创建 Environment 放到 config 上
        Environment.Builder environmentBuilder = new Environment.Builder(id)
          .transactionFactory(txFactory)
          .dataSource(dataSource);
        configuration.setEnvironment(environmentBuilder.build());
      }
    }
  }
}
// 处理 DatabaseProvider 
private void databaseIdProviderElement(XNode context) throws Exception {
  DatabaseIdProvider databaseIdProvider = null;
  if (context != null) {
    String type = context.getStringAttribute("type");
    // awful patch to keep backward compatibility
    if ("VENDOR".equals(type)) {
      type = "DB_VENDOR";
    }
    // 获取属性，就是数据库名称和别名对应
    Properties properties = context.getChildrenAsProperties();
    // 根据类型获取 databaseIdProvider 属性并设置属性
    databaseIdProvider = (DatabaseIdProvider) resolveClass(type).getDeclaredConstructor().newInstance();
    databaseIdProvider.setProperties(properties);
  }
  Environment environment = configuration.getEnvironment();
  if (environment != null && databaseIdProvider != null) {
    // 连接数据库获取数据名称并和上面读取的属性对比返回数据库别名，放到 config 上
    String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
    configuration.setDatabaseId(databaseId);
  }
}
// 注册 TypeHandler
private void typeHandlerElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 如果配置的是包则逐个扫描包
      if ("package".equals(child.getName())) {
        String typeHandlerPackage = child.getStringAttribute("name");
        typeHandlerRegistry.register(typeHandlerPackage);
      } else {
        // 获取相应的属性注册 typeHandler
        String javaTypeName = child.getStringAttribute("javaType");
        String jdbcTypeName = child.getStringAttribute("jdbcType");
        String handlerTypeName = child.getStringAttribute("handler");
        Class<?> javaTypeClass = resolveClass(javaTypeName);
        JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
        Class<?> typeHandlerClass = resolveClass(handlerTypeName);
        if (javaTypeClass != null) {
          if (jdbcType == null) {
            typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
          } else {
            typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
          }
        } else {
          typeHandlerRegistry.register(typeHandlerClass);
        }
      }
    }
  }
}
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 如果配置的是包则扫描包
      if ("package".equals(child.getName())) {
        String mapperPackage = child.getStringAttribute("name");
        configuration.addMappers(mapperPackage);
      } else {
        // 否则根据 resource， url 或 class 属性逐个解析配置
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        String mapperClass = child.getStringAttribute("class");
        if (resource != null && url == null && mapperClass == null) {
          ErrorContext.instance().resource(resource);
          InputStream inputStream = Resources.getResourceAsStream(resource);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url != null && mapperClass == null) {
          ErrorContext.instance().resource(url);
          InputStream inputStream = Resources.getUrlAsStream(url);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url == null && mapperClass != null) {
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          configuration.addMapper(mapperInterface);
        } else {
          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
        }
      }
      // 上面最终会调用两个方法分别是 XMLMapperBuilder 和 MapperAnnotationBuilder 其实就是一个处理类一个处理 xml 文件
      // 两者处理过程类似。只不过是前者从 xml 读取信息，后者从 注解上获取信息
    }
  }
}
```

```java
// XMLMapperBuilder
public void parse() {
  // 防止多次加载
  if (!configuration.isResourceLoaded(resource)) {
    // 解析 Mapper
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    // 绑定 Mapper 和 namespace
    bindMapperForNamespace();
  }

  // 解析因为异常没能完整解析的部分
  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
private void configurationElement(XNode context) {
  try {
    // 获取命名空间
    String namespace = context.getStringAttribute("namespace");
    if (namespace == null || namespace.equals("")) {
      throw new BuilderException("Mapper's namespace cannot be empty");
    }
    builderAssistant.setCurrentNamespace(namespace);
    // 解析 cache-ref 多个 Mapper 共享 cache 的时候配置
    cacheRefElement(context.evalNode("cache-ref"));
    cacheElement(context.evalNode("cache"));
    parameterMapElement(context.evalNodes("/mapper/parameterMap"));
    resultMapElements(context.evalNodes("/mapper/resultMap"));
    sqlElement(context.evalNodes("/mapper/sql"));
    // 生成 statement
    buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
  }
}
// 获取 Cache 的属性并生成一个  Cache 放到 config 上，这里也可以看出 cache 属性的默认值
private void cacheElement(XNode context) {
  if (context != null) {
    String type = context.getStringAttribute("type", "PERPETUAL");
    Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
    String eviction = context.getStringAttribute("eviction", "LRU");
    Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
    Long flushInterval = context.getLongAttribute("flushInterval");
    Integer size = context.getIntAttribute("size");
    boolean readWrite = !context.getBooleanAttribute("readOnly", false);
    boolean blocking = context.getBooleanAttribute("blocking", false);
    Properties props = context.getChildrenAsProperties();
    builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
  }
}
// 生成 statement
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
  for (XNode context : list) {
    final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
    try {
      statementParser.parseStatementNode();
    } catch (IncompleteElementException e) {
      configuration.addIncompleteStatement(statementParser);
    }
  }
}

// 生成 statement
public void parseStatementNode() {
  String id = context.getStringAttribute("id");
  String databaseId = context.getStringAttribute("databaseId");

  // 判断数据是否匹配
  if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
    return;
  }

  // 获取相关的属性
  String nodeName = context.getNode().getNodeName();
  SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
  boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
  boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
  boolean useCache = context.getBooleanAttribute("useCache", isSelect);
  boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

  // Include Fragments before parsing
  XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
  includeParser.applyIncludes(context.getNode());

  String parameterType = context.getStringAttribute("parameterType");
  Class<?> parameterTypeClass = resolveClass(parameterType);

  String lang = context.getStringAttribute("lang");
  LanguageDriver langDriver = getLanguageDriver(lang);

  // Parse selectKey after includes and remove them.
  processSelectKeyNodes(id, parameterTypeClass, langDriver);

  // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
  KeyGenerator keyGenerator;
  String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
  keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
  if (configuration.hasKeyGenerator(keyStatementId)) {
    keyGenerator = configuration.getKeyGenerator(keyStatementId);
  } else {
    keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                                               configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
      ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
  }

  SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
  StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
  Integer fetchSize = context.getIntAttribute("fetchSize");
  Integer timeout = context.getIntAttribute("timeout");
  String parameterMap = context.getStringAttribute("parameterMap");
  String resultType = context.getStringAttribute("resultType");
  Class<?> resultTypeClass = resolveClass(resultType);
  String resultMap = context.getStringAttribute("resultMap");
  String resultSetType = context.getStringAttribute("resultSetType");
  ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
  if (resultSetTypeEnum == null) {
    resultSetTypeEnum = configuration.getDefaultResultSetType();
  }
  String keyProperty = context.getStringAttribute("keyProperty");
  String keyColumn = context.getStringAttribute("keyColumn");
  String resultSets = context.getStringAttribute("resultSets");

  // 根据上面的配置生成一个 statement 并放到 config 上
  builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                                      fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                                      resultSetTypeEnum, flushCache, useCache, resultOrdered,
                                      keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

### 获取 `SqlSession`

```java
// DefaultSqlSessionFactory
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    // 从 Config 中获胜 environment，根据 environment 创建一个新的事务
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    // 根据 execType 获取 Executor，默认是 SIMPLE
    final Executor executor = configuration.newExecutor(tx, execType);
    // 把上面的信息挂到一个 DefaultSqlSession 上返回
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  // 根据 Executor Type 获取不同的 Executor
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  // 如果启用了 cache 则在原本的 Executor 上包装一层
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  // 调用 interceptor 链处理 Executor
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

### 获取代理对象

```java
// DefaultSqlSession
public <T> T getMapper(Class<T> type) {
  return configuration.getMapper(type, this);
}
```

```java
// Configuration
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

```java
// MapperRegistry
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 根据 mapper type 获取 MapperProxyFactory
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

```java
// MapperProxyFactory
public T newInstance(SqlSession sqlSession) {
  // 生成一个动态代理对象
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
protected T newInstance(MapperProxy<T> mapperProxy) {
  // 生成代理对象并返回
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```

### 执行方法

```java
// MapperProxy
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        // 如果是 Object 类中定义的方法则直接放行
        return method.invoke(this, args);
      } else if (method.isDefault()) {
        // 如果 interface 中定义的 default 方法同样放行
        if (privateLookupInMethod == null) {
          return invokeDefaultMethodJava8(proxy, method, args);
        } else {
          return invokeDefaultMethodJava9(proxy, method, args);
        }
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  	// 缓存方法
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

```java
// MapperMethod
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  switch (command.getType()) {
    case INSERT: {
      // 转换参数。具体流程在 MyBaits 的 SQL 映射中分析过，此处不再赘述
      Object param = method.convertArgsToSqlCommandParam(args);
      // 调用插入方法
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
        if (method.returnsOptional()
            && (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName()
                               + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```

#### 更新方法

```java
// DefaultSqlSession
// 对于 insert update delete 最终都使用这个方法
public int update(String statement, Object parameter) {
  try {
    dirty = true;
    // 从 config 中根据 ID 获取 MappedStatement
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.update(ms, wrapCollection(parameter));
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}

// 对 Collection 参数进行封装
private Object wrapCollection(final Object object) {
  if (object instanceof Collection) {
    StrictMap<Object> map = new StrictMap<>();
    map.put("collection", object);
    if (object instanceof List) {
      map.put("list", object);
    }
    return map;
  } else if (object != null && object.getClass().isArray()) {
    StrictMap<Object> map = new StrictMap<>();
    map.put("array", object);
    return map;
  }
  return object;
}
```

```java
// CachingExecutor
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
  // 刷新缓存
  flushCacheIfRequired(ms);
  // 使用 Executor 真正的去 update
  return delegate.update(ms, parameterObject);
}
```

```java
// BaseExecutor
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  // 清理一级缓存
  clearLocalCache();
  return doUpdate(ms, parameter);
}
```

```java
// SimpleExecutor
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
  Statement stmt = null;
  try {
    // 获取 configuration
    Configuration configuration = ms.getConfiguration();
    // 获取 StatementHandler
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    // Prepare Statement
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.update(stmt);
  } finally {
    closeStatement(stmt);
  }
}
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  // 获取连接
  Connection connection = getConnection(statementLog);
  // prepare statement
  stmt = handler.prepare(connection, transaction.getTimeout());
  // 设置参数
  handler.parameterize(stmt);
  return stmt;
}
```

```java
// Configuration
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  // 新建一个 StatementHandler
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  // 使用 interceptor 修改 statement
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
```

```java
// RoutingStatementHandler
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  // 根据 statement 的 type 创建不同的 StatementHandler
  switch (ms.getStatementType()) {
    case STATEMENT:
      delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case PREPARED:
      delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case CALLABLE:
      delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    default:
      throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
  }
}
```

```java
// BaseStatementHandler
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
  ErrorContext.instance().sql(boundSql.getSql());
  Statement statement = null;
  try {
    // 填充 statement
    statement = instantiateStatement(connection);
    // 设置超时时间
    setStatementTimeout(statement, transactionTimeout);
    // 设置获取 size
    setFetchSize(statement);
    return statement;
  } catch (SQLException e) {
    closeStatement(statement);
    throw e;
  } catch (Exception e) {
    closeStatement(statement);
    throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
  }
}
```

```java
// DefaultParameterHandler
public void setParameters(PreparedStatement ps) {
  ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
  // 获取参数的列表
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings != null) {
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        // 获取属性名
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          // 从参数中获取 value
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        // 获取 TypeHandler
        TypeHandler typeHandler = parameterMapping.getTypeHandler();
        JdbcType jdbcType = parameterMapping.getJdbcType();
        // 获取为 null 的时候对应的 jdbcType
        if (value == null && jdbcType == null) {
          jdbcType = configuration.getJdbcTypeForNull();
        }
        try {
          // 根据不同的 JDBC Type 调用各自的方法设置 value
          typeHandler.setParameter(ps, i + 1, value, jdbcType);
        } catch (TypeException | SQLException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        }
      }
    }
  }
}
```

```java
// PreparedStatementHandler
public int update(Statement statement) throws SQLException {
  // 转换 statement 并执行
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  // 获取更新的 Count
  int rows = ps.getUpdateCount();
  Object parameterObject = boundSql.getParameterObject();
  // 获取主键设置到参数对象上
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
  return rows;
}
```

#### 查询方法

返回单个元素和返回一个 List 的执行方法几乎一致，只是单个元素在返回前从 List 中 取出第一个返回。对于返回 Map 略有不同，主要区别是在 List  查询之后会根据设置的条件进行 List 转 Map的操作。

```java
// DefaultSqlSession
public <T> T selectOne(String statement, Object parameter) {
  // 调用 select list 取第一个返回，数量超过 1 则抛出异常
  List<T> list = this.selectList(statement, parameter);
  if (list.size() == 1) {
    return list.get(0);
  } else if (list.size() > 1) {
    throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
  } else {
    return null;
  }
}

public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    // 从 config 中获取 MappedStatement
    MappedStatement ms = configuration.getMappedStatement(statement);
    // 执行查询方法
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

```java
// CachingExecutor
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  // 获取关联的 SQL
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  // 根据传入的参数生成一个 cacheKey
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
  // 从 Statement 获取 cache
  Cache cache = ms.getCache();
  if (cache != null) {
    // 刷新缓存
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      // 从 TransactionalCacheManager 根据 cache 和 key 获取结果
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        // 将查询结果放到 cache 中
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

```java
// MappedStatement
public BoundSql getBoundSql(Object parameterObject) {
  // 根据参数信息生成 BoundSql
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
  }

  // check for nested result maps in parameter mappings (issue #30)
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) {
        hasNestedResultMaps |= rm.hasNestedResultMaps();
      }
    }
  }

  return boundSql;
}
```

```java
// BaseExecutor
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;
    // 尝试从 cache 中获取
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      // 处理从 cache 中获取的结果
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // issue #601
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // issue #482
      clearLocalCache();
    }
  }
  return list;
}

private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    // 执行 Query 方法
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    localCache.removeObject(key);
  }
  // 把结果放到缓存中
  localCache.putObject(key, list);
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

```java
// SimpleExecutor
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    // 获取 Config
    Configuration configuration = ms.getConfiguration();
    // 获取 StatementHandler
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    // 准备 statement 设置参数
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行查询
    return handler.query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

```java
// PreparedStatementHandler
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  // 执行 sql
  ps.execute();
  // 处理结果
  return resultSetHandler.handleResultSets(ps);
}
```

```java
// DefaultResultSetHandler
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

  final List<Object> multipleResults = new ArrayList<>();

  int resultSetCount = 0;
  // 获取 result
  ResultSetWrapper rsw = getFirstResultSet(stmt);

  // 获取 result map
  List<ResultMap> resultMaps = mappedStatement.getResultMaps();
  int resultMapCount = resultMaps.size();
  validateResultMapsCount(rsw, resultMapCount);
  while (rsw != null && resultMapCount > resultSetCount) {
    // 遍历获取 resultMap
    ResultMap resultMap = resultMaps.get(resultSetCount);
    // 处理结果
    handleResultSet(rsw, resultMap, multipleResults, null);
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
  }

  String[] resultSets = mappedStatement.getResultSets();
  if (resultSets != null) {
    while (rsw != null && resultSetCount < resultSets.length) {
      ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
      if (parentMapping != null) {
        String nestedResultMapId = parentMapping.getNestedResultMapId();
        ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
        handleResultSet(rsw, resultMap, null, parentMapping);
      }
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
  }

  return collapseSingleResultList(multipleResults);
}

private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
  try {
    if (parentMapping != null) {
      handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
    } else {
      if (resultHandler == null) {
        DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
        handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
        multipleResults.add(defaultResultHandler.getResultList());
      } else {
        handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
      }
    }
  } finally {
    // issue #228 (close resultsets)
    closeResultSet(rsw.getResultSet());
  }
}

public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
  if (resultMap.hasNestedResultMaps()) {
    ensureNoRowBounds();
    checkResultHandler();
    handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  } else {
    handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  }
}

private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
  DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
  ResultSet resultSet = rsw.getResultSet();
  skipRows(resultSet, rowBounds);
  while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
    // Resolve Discriminated ResultMap
    ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
    // 获取结果
    Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
    // 存储结果
    storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
  }
}

private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
  final ResultLoaderMap lazyLoader = new ResultLoaderMap();
  // 创建结果对象
  Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
  if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
    final MetaObject metaObject = configuration.newMetaObject(rowValue);
    boolean foundValues = this.useConstructorMappings;
    // 自动赋值
    if (shouldApplyAutomaticMappings(resultMap, false)) {
      foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
    }
    // 根据自定义 mapping 中获取设置属性值
    foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
    foundValues = lazyLoader.size() > 0 || foundValues;
    rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
  }
  return rowValue;
}
```







