## Spring Boot 自动配置

众所周知 Spring Boot 极大的减少了 Spring 繁琐的配置，采用约定大于配置的策略。也是基于此 Spring Boot 有很多自动配置的逻辑，下面简单理一下相关的代码。

首先对于 Spring Boot 应用主启动类上要标注 `@SpringBootApplication` 如下所示，那么这个注解为会 Spring Boot 添加哪些配置的呢？下面可以看一下相关的源码。

```java
@SpringBootApplication
public class MyMainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyMainApplication.class);
    }
}
```

引入如下的注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 主要是一个 @Configuration 的注解
@SpringBootConfiguration  
// 开启自动配置
@EnableAutoConfiguration
// 开启包扫描并设置了一些过滤条件
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

```java
// 标注当前类是 Config 类
@Configuration
public @interface SpringBootConfiguration {
```

```java
// 自动配置包
@AutoConfigurationPackage
// 为容器引入 AutoConfigurationImportSelector
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 为容器引入 AutoConfigurationPackages.Registrar
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
```

通过 `AutoConfigurationImportSelector` 为容器中注入 Bean，最终通过其内部类 `AutoConfigurationGroup` 的 `selectImports` 为容器中导入 Bean。用于导入预设的所有 AutoConfiguration 的类并进行自动配置。会从 `spring.factories` 文件中获取 `EnableAutoConfiguration` 的值作为自动配置类导入到容器中， AutoConfiguration 类的元信息定义在 `spring-autoconfigure-metadata.properties` 文件中。

==ToDo:  增加调用流程==

```java
// AutoConfigurationImportSelector.AutoConfigurationGroup#selectImports
@Override
public Iterable<Entry> selectImports() {
    // Autoconfiguration 类如下图所示
    if (this.autoConfigurationEntries.isEmpty()) {
        return Collections.emptyList();
    }
    // 获取需要排除的 Config
    Set<String> allExclusions = this.autoConfigurationEntries.stream()
        .map(AutoConfigurationEntry::getExclusions).flatMap(Collection::stream).collect(Collectors.toSet());
    // 获取需要处理的 Config
    Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
        .map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
        .collect(Collectors.toCollection(LinkedHashSet::new));
    // 去除所有排除的 Config
    processedConfigurations.removeAll(allExclusions);

    // 排序 AutoConfiguration 生成 Entry 的 List 返回
    return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
        .map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
        .collect(Collectors.toList());
}
```

![AutoConfiguration 类]( http://img.sangzhenya.com/Snipaste_2019-11-24_19-24-49.png )

![AutoConfiguration 配置文件]( http://img.sangzhenya.com/Snipaste_2019-11-24_19-36-17.png )

通过 `AutoConfigurationPackages.Registrar` 为容器中注入 Bean

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 元信息如下图所示，获取到的包名即主配置类所在的包名
        // 目的就是讲主配置类所在包下的所有的类注入到容器中
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.singleton(new PackageImport(metadata));
    }

}
```

```java
// 将传入的 package name 注册到容器中
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
    if (registry.containsBeanDefinition(BEAN)) {
        BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
        ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
        constructorArguments.addIndexedArgumentValue(0, addBasePackages(constructorArguments, packageNames));
    }
    else {
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(BasePackages.class);
        beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
        beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        registry.registerBeanDefinition(BEAN, beanDefinition);
    }
}
```

![元信息]( http://img.sangzhenya.com/Snipaste_2019-11-24_19-09-16.png )

上面已经知道自动配置主要是依赖于启动的时候导入的 AutoConfiguration 类，下面分析以下 AutoConfiguration 的流程，以 `HttpEncodingAutoConfiguration` 为例：

```java
// 标注配置类
@Configuration(proxyBeanMethods = false)
// 启动 Configuration Properties 功能，即启动 HttpProperties 从 properties 中取值和 Bean 属性绑定
@EnableConfigurationProperties(HttpProperties.class)
// 启动该配置的条件 Web 环境，有 CharacterEncodingFilter 类，
// 无 spring.http.encoding.enabled 或值为 true
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnClass(CharacterEncodingFilter.class)
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {

    // 获取属性值
	private final HttpProperties.Encoding properties;

	public HttpEncodingAutoConfiguration(HttpProperties properties) {
		this.properties = properties.getEncoding();
	}

    // 给容器中添加一个 CharacterEncodingFilter Bean
	@Bean
    // 只有在容器中没有该 Bean 的时候才会添加
	@ConditionalOnMissingBean
	public CharacterEncodingFilter characterEncodingFilter() {
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
        // 从 properties 中获取值
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
		return filter;
	}

    // 为容器添加 LocaleCharsetMappingsCustomizer Bean
    // 用于帮助使用者自定义 Configuration
	@Bean
	public LocaleCharsetMappingsCustomizer localeCharsetMappingsCustomizer() {
		return new LocaleCharsetMappingsCustomizer(this.properties);
	}

	private static class LocaleCharsetMappingsCustomizer
			implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>, Ordered {

		private final HttpProperties.Encoding properties;

		LocaleCharsetMappingsCustomizer(HttpProperties.Encoding properties) {
			this.properties = properties;
		}

		@Override
		public void customize(ConfigurableServletWebServerFactory factory) {
			if (this.properties.getMapping() != null) {
				factory.setLocaleCharsetMappings(this.properties.getMapping());
			}
		}

		@Override
		public int getOrder() {
			return 0;
		}

	}

}
```

```java
// 配置的开头都是以 spring.http 开头
@ConfigurationProperties(prefix = "spring.http")
public class HttpProperties {
    private boolean logRequestDetails;
    public static class Encoding {

		public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;
```

