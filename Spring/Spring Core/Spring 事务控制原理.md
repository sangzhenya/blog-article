## Spring 事务控制原理

使用 Spring 注解的方式启用事务，首先要使用 `@EnableTransactionManagement` 注解开启全局事务，事务需要交给具体的 TransactionManager 进行管理，所以需要向容器中添加相应的 Bean，如下所示

```java
 @Bean
 public PlatformTransactionManager transactionManager() throws PropertyVetoException {
 return new DataSourceTransactionManager(dataSource());
 }
```

最后需要在具体需要添加事务的方法上加上 `@Transactional` 注解。

Spring 的事务隔离级别有以下几种：

1. `TransactionDefinition.ISOLATION_DEFAULT`（使用后端数据库默认的隔离级别）
2. `TransactionDefinition.ISOLATION_READ_UNCOMMITTED`
3. `TransactionDefinition.ISOLATION_READ_COMMITTED`
4. `TransactionDefinition.ISOLATION_REPEATABLE_READ`
5. `TransactionDefinition.ISOLATION_SERIALIZABLE`

Spring 的事务传播行为以下几种：

1. `TransactionDefinition.PROPAGATION_REQUIRED`： 有则加入，没有则新建事务
2. `TransactionDefinition.PROPAGATION_SUPPORTS`：有则加入，没有则以非事务模式运行
3. `TransactionDefinition.PROPAGATION_MANDATORY`： 有则加入，没有则抛出异常
4. `TransactionDefinition.PROPAGATION_REQUIRES_NEW`： 将当前事务挂起，新建一个事务
5. `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`：将当前事务挂起，以非事务运行
6. `TransactionDefinition.PROPAGATION_NEVER`：当前有事务则抛出异常，否则以非事务运行
7. `TransactionDefinition.PROPAGATION_NESTED` 嵌套事务



具体 Spring 实现事务的原理如下：

在 `@EnableTransactionManagement` 注解中引入了 `TransactionManagementConfigurationSelector` 类，该类如下：

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
        // 根据不同的模式导入不同的 config，默认是 PROXY 模式
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}

	private String determineTransactionAspectClass() {
		return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
				TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
				TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
	}

}
```

