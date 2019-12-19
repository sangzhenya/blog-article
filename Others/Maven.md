## Maven 入门

### 构建的各个环节

1. 清理：将以前编译得到的旧的 class 字节码文件删除，为下一次编译做准备
2. 编译：将 Java 源程序编译成 class 字节码文件
3. 测试：自动测试，自动调用 junit 程序
4. 报告：测试程序执行的结果
5. 打包：动态 Web工程打 war 包，Java 工程打 jar 包
6. 安装：将打包得到的文件复制到 ”仓库“ 中指定的位置
7. 部署：将打包后的文件部署运行。



### 核心概念

1. 约定的目录结构
2. POM
3. 坐标
4. 依赖
5. 仓库
6. 生命周期/插件/目标
7. 继承
8. 聚合

#### 目录结构

基本的目录结构如图所示：

![Maven 目录结构](https://i.loli.net/2019/06/16/5d05dcc02cf4050805.png)

1. 根目录：工程名称
2. src 目录：源码
3. pom.xml 文件： Maven 工程核心文件
4. main 目录：存放主程序的地方
5. test 目录：存放测试程序的地方
6. java 目录：存放 Java 源文件的地方
7. resources 目录：存放框架或其他工具配置文件的地方

**约定 > 配置 > 编码**

#### POM 文件

Project Object Model 项目对象模型，是核心配置文件，与构建的过程一切设置都在这个文件进行配置。

一个简单地 pom.xml 文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.xinyue</groupId>
    <artifactId>demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <name>demo</name>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>5.5.0-M1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

#### 坐标

Maven 中使用 groupId, artifactId, version 三个向量的坐标定位一个 maven 工厂

1. **groupId:** 公司或组织的域名倒序 + 项目名称
2. **artifactId:** 模块名称
3. **version:** 版本

和本地的对应关系如下：

```xml
<!-- Spring Core 的坐标 -->
<groupId>org.springframework</groupId>
<artifactId>spring-core</artifactId>
<version>5.1.5.RELEASE</version>

<!-- Spring Core 本地仓库对应的位置 -->
org\springframework\spring-core\5.1.5.RELEASE
```

#### 依赖

Maven 解析依赖信息的时候会从本地仓库中查找被依赖的 jar 包。*自己开发的工程可以通过 `mvn install` 安装到仓库中*

依赖范围：

|                    | compile (默认值) | test  | provided    |
| ------------------ | ---------------- | ----- | ----------- |
| 是否参与打包       | 是               | 是    | 是          |
| 对测试程序是否有效 | 是               | 是    | 是          |
| 是否参与打包       | 是               | 否    | 否          |
| 是否参与部署       | 是               | 否    | 否          |
| 典型例子           | spring-core      | junit | servlet-api |

依赖具有传递性，不需要在每个模块工程中都重复声明。但是只有依赖范围是 compile 的依赖才会传递。另外依赖冲突的问题，不同版本的依赖 先声明优先，最短路径优先。

下面是一个简单的依赖关系，我们项目其实仅仅引入了 spring-core 和 junit-jupiter-api，而红色框中的依赖则是通过依赖传递过来的。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.1.8.RELEASE</version>
    </dependency>

    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.5.0-M1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```



![简单的依赖关系](https://i.loli.net/2019/06/16/5d05ed633369844162.png)

依赖也可进行排除，如下所示：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.1.8.RELEASE</version>
        <exclusions>
            <exclusion>
                <artifactId>spring-jcl</artifactId>
                <groupId>org.springframework</groupId>
            </exclusion>
        </exclusions>
    </dependency>

    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.5.0-M1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

![依赖排除](https://i.loli.net/2019/06/16/5d05ee097c89475143.png)

#### 仓库

分类：**本地仓库**和**远程仓库**两类。远程仓库可以分为：**私服**，**中央仓库**，**中央仓库镜像**三种。

保存内容：1 Maven 自身需要的插件； 2 第三方框架或工具的 jar 包；3 我们自己开发的 Maven 工程。

#### 生命周期/插件/目标

各个构建环节执行的顺序只能按照正确的顺序执行，不能乱序。Maven 的核心程序定义了抽象的生命周期，生命周期中各个阶段的具体任务是又插件完成的。生命周期如下：

1. Clean Lifecycle: 在进行真正的构建之前进行一些清理工作
   1. pre-clean: 直线一些需要在 clean 之前完成的工作
   2. clean: 移除所有上一次构建生成的文件
   3. post-clean: 执行一些需要在 clean 之后完成的工作
2. Default Lifecycle：构建和核心部分，编译，测试，打包，安装，部署等等
   1. validate
   2. generate-sources
   3. process-sources
   4. generate-resources
   5. process-resouces  
   6. compile  
   7. process-classes
   8. generate-test-sources
   9. process-test-sources
   10. generate-test-resouces
   11. process-test-resouces
   12. test-compile
   13. process-test-classes
   14. test
   15. prepare-package
   16. pre-integration-test
   17. integration-test
   18. post-integration-test
   19. verify
   20. install
   21. deploy
3. Site Lifecycle：生成项目报告，站点，发布站点。
   1. pre-site: 执行一些需要在生成站点文档之前工作
   2. site：生成项目的站点文件
   3. post-site：执行一些需要在生成站点文档之后的工作，并且为部署做准备
   4. site-deploy：将生成的站点文档部署到特定的服务器上

为了实现更好的自动化构建，执行生命周期的各个阶段都是从生命周期的最初位置开始执行的。

生命周期仅仅定义了各个阶段要执行的任务是什么，**各个阶段和插件的目标是对应的，相似的目标由特定的插件来完成。**

#### 继承

为了统一管理各个模块中不能被传递的依赖，将这些依赖统一抽取到父工程中，在父工程中统一设置版本，在子工程中生命依赖不含版本。

父工程 pom 文件设置简单如下：

```xml
<groupId>com.xinyue.demo</groupId>
<artifactId>parent</artifactId>
<version>0.0.1</version>
<packaging>pom</packaging>
<dependencyManagement>
    <dependencies>
    	<dependency>
        	<groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>5.5.0-M1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

子工程 pom 文件设置简单如下：

```xml
<parent>
	<groupID>com.xinyue.demo</groupID>
    <artifactId>parent</artifactId>
    <version>0.0.1</version>
    
    <relativePath>../parent/pom.xml</relativePath>
</parent>
<denpencies>
	<dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <scope>test</scope>
    </dependency>
</denpencies>
```

`dependencyManagement` 元素技能让子模块继承到父模块的依赖配置，又能保证模块依赖使用的灵活性。在 `dependencyManagement` 元素下声明的依赖不会实际引入，而是能够约束 `dependencies` 下的依赖使用。

#### 聚合

简单来说就是一件安装各个模块，可以在一个总的聚合工程中配置各个参与的模块。例如在父工程中可以如下配置：

```xml
<modules>
	<module>../module1</module>
    <module>../module2</module>
    <module>../module3</module>
</modules>
```



### Spring Properties

可以通过 `properties` 元素用户自定义一个或多个 Maven属性。然后在 POM 的其他地方使用 `${属性名称}` 的方式引用该属性，主要是为了消除重复。Maven 可以使用以下 6 种属性：

1. 内置属性

   ```shell
   ${basedir}	# 项目根目录，即包含 pom.xml 的目录
   ${version}	# 项目的版本
   ```

2. POM 属性

   ```shell
   ${project.groupId}			# group id
   ${project.artifactId}		# artifact id
   ${project.version}			# version
   ${project.build.finalName}	# artifactId-version
   ${project.build.sourceDirectory}	# 项目主目录  /src/main/java/
   ${project.build.testSourceDirectory}	# 测试源码目录 /scr/test/java/
   ${project.build.directory}			# 构建目录 /target
   ${project.outputDirectory}			# 主代码编译输出目录 /target/classes/
   ${project.testOutputDirectory}		# 测试代码编译输出目录 /target/test-classess/
   ```

3. 自定义属性

   ```xml
   <properties>
       <!-- 自定义属性 -->
       <customized.pro>demo</customized.pro1>
   </properties>
   ```

4. Setting 属性

   ```shell
   # 取自 settings.xml
   ${settings.localRepository}
   ```

5. Java 系统属性

   ```shell
   # 可以使用 mvn help: system 查看所有的 Java 系统属性
   ${user.home}	# 用户主目录
   ```

6. 环境变量属性

   ```shell
   # 可以使用 mvn help: system 查看所有的 环境变量属性
   ${env.JAVA_HOME}	# JAVA_HOME 环境变量的值
   ```

   

### 基本的 Maven 命令

执行与构建有关的 Maven 命令必须进入 pom.xml 所在的目录。包括编译，测试，打包。

常用的有以下：

```shell
mvn clean		# 清理
mvn compile		# 编译主程序
mvn test-compile	# 编译测试程序
mvn test		# 执行测试
mvn package		# 打包
mvn install		# 安装
mvn site		# 生成站点文件
```



### Settings 设置

摘录自：[Maven之（六）setting.xml配置文件详解](https://blog.csdn.net/u012152619/article/details/51485152)

```xml
<?xml version="1.0" encoding="UTF-8"?>

<settings   xmlns="http://maven.apache.org/POM/4.0.0"  
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
	  
	<!--本地仓库。该值表示构建系统本地仓库的路径。其默认值为${user.home}/.m2/repository。  -->
	<localRepository>usr/local/maven</localRepository>
	<!--Maven是否需要和用户交互以获得输入。如果Maven需要和用户交互以获得输入，则设置成true，反之则应为false。默认为true。 -->
	<interactiveMode>true</interactiveMode>
	<!--Maven是否需要使用plugin-registry.xml文件来管理插件版本。  -->
	<!--如果设置为true，则在{user.home}/.m2下需要有一个plugin-registry.xml来对plugin的版本进行管理  -->
	<!--默认为false。 -->
	<usePluginRegistry>false</usePluginRegistry>
	<!--表示Maven是否需要在离线模式下运行。如果构建系统需要在离线模式下运行，则为true，默认为false。  -->
	<!--当由于网络设置原因或者安全因素，构建服务器不能连接远程仓库的时候，该配置就十分有用。  -->
	<offline>false</offline>
	<!--当插件的组织Id（groupId）没有显式提供时，供搜寻插件组织Id（groupId）的列表。  -->
	<!--该元素包含一个pluginGroup元素列表，每个子元素包含了一个组织Id（groupId）。  -->
	<!--当我们使用某个插件，并且没有在命令行为其提供组织Id（groupId）的时候，Maven就会使用该列表。  -->
	<!--默认情况下该列表包含了org.apache.maven.plugins。  -->
	<pluginGroups>
		<!--plugin的组织Id（groupId）  -->
		<pluginGroup>org.codehaus.mojo</pluginGroup>
	</pluginGroups>
	<!--用来配置不同的代理，多代理profiles可以应对笔记本或移动设备的工作环境：通过简单的设置profile id就可以很容易的更换整个代理配置。  -->
	<proxies>
		<!--代理元素包含配置代理时需要的信息 -->
		<proxy>
			<!--代理的唯一定义符，用来区分不同的代理元素。 -->
			<id>myproxy</id>
			<!--该代理是否是激活的那个。true则激活代理。当我们声明了一组代理，而某个时候只需要激活一个代理的时候，该元素就可以派上用处。  -->
			<active>true</active>
			<!--代理的协议。 协议://主机名:端口，分隔成离散的元素以方便配置。 -->
			<protocol>http://…</protocol>
			<!--代理的主机名。协议://主机名:端口，分隔成离散的元素以方便配置。   -->
			<host>proxy.somewhere.com</host>
			<!--代理的端口。协议://主机名:端口，分隔成离散的元素以方便配置。  -->
			<port>8080</port>
			 <!--代理的用户名，用户名和密码表示代理服务器认证的登录名和密码。  -->
			<username>proxyuser</username>
			<!--代理的密码，用户名和密码表示代理服务器认证的登录名和密码。  -->
			<password>somepassword</password>
			<!--不该被代理的主机名列表。该列表的分隔符由代理服务器指定；例子中使用了竖线分隔符，使用逗号分隔也很常见。 -->
			<nonProxyHosts>*.google.com|ibiblio.org</nonProxyHosts>
		</proxy>
	</proxies>
	<!--配置服务端的一些设置。一些设置如安全证书不应该和pom.xml一起分发。这种类型的信息应该存在于构建服务器上的settings.xml文件中。 -->
	<servers>
		<!--服务器元素包含配置服务器时需要的信息  -->
		<server>
			<!--这是server的id（注意不是用户登陆的id），该id与distributionManagement中repository元素的id相匹配。 -->
			<id>server001</id>
			<!--鉴权用户名。鉴权用户名和鉴权密码表示服务器认证所需要的登录名和密码。  -->
			<username>my_login</username>
			<!--鉴权密码 。鉴权用户名和鉴权密码表示服务器认证所需要的登录名和密码。  -->
			<password>my_password</password>
			<!--鉴权时使用的私钥位置。和前两个元素类似，私钥位置和私钥密码指定了一个私钥的路径（默认是/home/hudson/.ssh/id_dsa）以及如果需要的话，一个密钥 -->
			<!--将来passphrase和password元素可能会被提取到外部，但目前它们必须在settings.xml文件以纯文本的形式声明。  -->
			<privateKey>${usr.home}/.ssh/id_dsa</privateKey>
			<!--鉴权时使用的私钥密码。 -->
			<passphrase>some_passphrase</passphrase>
			<!--文件被创建时的权限。如果在部署的时候会创建一个仓库文件或者目录，这时候就可以使用权限（permission）。-->
			<!--这两个元素合法的值是一个三位数字，其对应了unix文件系统的权限，如664，或者775。  -->
			<filePermissions>664</filePermissions>
			<!--目录被创建时的权限。  -->
			<directoryPermissions>775</directoryPermissions>
			<!--传输层额外的配置项  -->
			<configuration></configuration>
		</server>
	</servers>
	<!--为仓库列表配置的下载镜像列表。  -->
	<mirrors>
		<!--给定仓库的下载镜像。  -->
		<mirror>
			<!--该镜像的唯一标识符。id用来区分不同的mirror元素。  -->
			<id>planetmirror.com</id>
			<!--镜像名称  -->
			<name>PlanetMirror Australia</name>
			<!--该镜像的URL。构建系统会优先考虑使用该URL，而非使用默认的服务器URL。  -->
			<url>http://downloads.planetmirror.com/pub/maven2</url>
			<!--被镜像的服务器的id。例如，如果我们要设置了一个Maven中央仓库（http://repo1.maven.org/maven2）的镜像，-->
			<!--就需要将该元素设置成central。这必须和中央仓库的id central完全一致。 -->
			<mirrorOf>central</mirrorOf>
		</mirror>
	</mirrors>
	<!--根据环境参数来调整构建配置的列表。settings.xml中的profile元素是pom.xml中profile元素的裁剪版本。-->
	<!--它包含了id，activation, repositories, pluginRepositories和 properties元素。-->
	<!--这里的profile元素只包含这五个子元素是因为这里只关心构建系统这个整体（这正是settings.xml文件的角色定位），而非单独的项目对象模型设置。-->
	<!--如果一个settings中的profile被激活，它的值会覆盖任何其它定义在POM中或者profile.xml中的带有相同id的profile。  -->
	<profiles>
		<!--根据环境参数来调整的构件的配置 -->
		<profile>
			<!--该配置的唯一标识符。  -->
			<id>test</id>
			<!--自动触发profile的条件逻辑。Activation是profile的开启钥匙。-->
			<!--如POM中的profile一样，profile的力量来自于它能够在某些特定的环境中自动使用某些特定的值；这些环境通过activation元素指定。-->
			<!--activation元素并不是激活profile的唯一方式。settings.xml文件中的activeProfile元素可以包含profile的id。-->
			<!--profile也可以通过在命令行，使用-P标记和逗号分隔的列表来显式的激活（如，-P test）。 -->
			<activation>
				<!--profile默认是否激活的标识 -->
				<activeByDefault>false</activeByDefault>
				<!--activation有一个内建的java版本检测，如果检测到jdk版本与期待的一样，profile被激活。 -->
				<jdk>1.7</jdk>
				<!--当匹配的操作系统属性被检测到，profile被激活。os元素可以定义一些操作系统相关的属性。 -->
				<os>
					<!--激活profile的操作系统的名字  -->
					<name>Windows XP</name>
					<!--激活profile的操作系统所属家族(如 'windows')   -->
					<family>Windows</family>
					<!--激活profile的操作系统体系结构   -->
					<arch>x86</arch>
					<!--激活profile的操作系统版本 -->
					<version>5.1.2600</version>
				</os>
				<!--如果Maven检测到某一个属性（其值可以在POM中通过${名称}引用），其拥有对应的名称和值，Profile就会被激活。-->
				<!--如果值字段是空的，那么存在属性名称字段就会激活profile，否则按区分大小写方式匹配属性值字段 -->
				<property>
					<!--激活profile的属性的名称 -->
					<name>mavenVersion</name>
					<!--激活profile的属性的值  -->
					<value>2.0.3</value>
				</property>
				<!--提供一个文件名，通过检测该文件的存在或不存在来激活profile。missing检查文件是否存在，如果不存在则激活profile。-->
				<!--另一方面，exists则会检查文件是否存在，如果存在则激活profile。 -->
				<file>
					<!--如果指定的文件存在，则激活profile。  -->
					<exists>/usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/</exists>
					<!--如果指定的文件不存在，则激活profile。 -->
					<missing>/usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/</missing>
				</file>
			</activation>
			 <!--对应profile的扩展属性列表。Maven属性和Ant中的属性一样，可以用来存放一些值。这些值可以在POM中的任何地方使用标记${X}来使用，这里X是指属性的名称。-->
			<!--属性有五种不同的形式，并且都能在settings.xml文件中访问。   -->
			<!--1. env.X: 在一个变量前加上"env."的前缀，会返回一个shell环境变量。例如,"env.PATH"指代了$path环境变量（在Windows上是%PATH%）。  --> 
			<!--2. project.x：指代了POM中对应的元素值。      -->
			<!--3. settings.x: 指代了settings.xml中对应元素的值。   -->
			<!--4. Java System Properties: 所有可通过java.lang.System.getProperties()访问的属性都能在POM中使用该形式访问，   -->
			<!--   如/usr/lib/jvm/java-1.6.0-openjdk-1.6.0.0/jre。      -->
			<!--5. x: 在<properties/>元素中，或者外部文件中设置，以${someVar}的形式使用。  -->
			<properties>
				<!-- 如果这个profile被激活，那么属性${user.install}就可以被访问了 -->
				<user.install>usr/local/winner/jobs/maven-guide</user.install>
			</properties>
			<!--远程仓库列表，它是Maven用来填充构建系统本地仓库所使用的一组远程项目。  -->
			<repositories>
				<!--包含需要连接到远程仓库的信息  -->
				<repository>
					<!--远程仓库唯一标识 -->
					<id>codehausSnapshots</id>
					<!--远程仓库名称  -->
					<name>Codehaus Snapshots</name>
					<!--如何处理远程仓库里发布版本的下载 -->
					<releases>
						<!--true或者false表示该仓库是否为下载某种类型构件（发布版，快照版）开启。   -->
						<enabled>false</enabled>
						<!--该元素指定更新发生的频率。Maven会比较本地POM和远程POM的时间戳。这里的选项是：-->
						<!--always（一直），daily（默认，每日），interval：X（这里X是以分钟为单位的时间间隔），或者never（从不）。  -->
						<updatePolicy>always</updatePolicy>
						<!--当Maven验证构件校验文件失败时该怎么做:-->
					    <!--ignore（忽略），fail（失败），或者warn（警告）。 -->
						<checksumPolicy>warn</checksumPolicy>
					</releases>
					<!--如何处理远程仓库里快照版本的下载。有了releases和snapshots这两组配置，POM就可以在每个单独的仓库中，为每种类型的构件采取不同的策略。-->
					<!--例如，可能有人会决定只为开发目的开启对快照版本下载的支持。参见repositories/repository/releases元素 -->
					<snapshots>
						<enabled />
						<updatePolicy />
						<checksumPolicy />
					</snapshots>
					<!--远程仓库URL，按protocol://hostname/path形式  -->
					<url>http://snapshots.maven.codehaus.org/maven2</url>
					<!--用于定位和排序构件的仓库布局类型-可以是default（默认）或者legacy（遗留）。-->
					<!--Maven 2为其仓库提供了一个默认的布局；然而，Maven 1.x有一种不同的布局。我们可以使用该元素指定布局是default（默认）还是legacy（遗留）。  -->
					<layout>default</layout>
				</repository>
			</repositories>
			<!--发现插件的远程仓库列表。仓库是两种主要构件的家。第一种构件被用作其它构件的依赖。这是中央仓库中存储的大部分构件类型。另外一种构件类型是插件。-->
			<!--Maven插件是一种特殊类型的构件。由于这个原因，插件仓库独立于其它仓库。pluginRepositories元素的结构和repositories元素的结构类似。-->
			<!--每个pluginRepository元素指定一个Maven可以用来寻找新插件的远程地址。 -->
			<pluginRepositories>
				<!--包含需要连接到远程插件仓库的信息.参见profiles/profile/repositories/repository元素的说明 -->
				<pluginRepository>
					<releases>
						<enabled />
						<updatePolicy />
						<checksumPolicy />
					</releases>
					<snapshots>
						<enabled />
						<updatePolicy />
						<checksumPolicy />
					</snapshots>
					<id />
					<name />
					<url />
					<layout />
				</pluginRepository>
			</pluginRepositories>
			<!--手动激活profiles的列表，按照profile被应用的顺序定义activeProfile。 该元素包含了一组activeProfile元素，每个activeProfile都含有一个profile id。-->
			<!--任何在activeProfile中定义的profile id，不论环境设置如何，其对应的 profile都会被激活。-->
			<!--如果没有匹配的profile，则什么都不会发生。例如，env-test是一个activeProfile，则在pom.xml（或者profile.xml）中对应id的profile会被激活。-->
			<!--如果运行过程中找不到这样一个profile，Maven则会像往常一样运行。  -->
			<activeProfiles>
				<activeProfile>env-test</activeProfile>
			</activeProfiles>
		</profile>
	</profiles>  
</settings>
```



