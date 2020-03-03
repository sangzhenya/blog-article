## JPA 基本注解

JDBC 是 Sun公司提供了一套访问数据库的接口规范，数据库厂商必须遵守Sun公司的规范，提供访问自己数据库服务器API即驱动；JDBC只是一些接口，需要数据库厂商提供相应的实现类即驱动，如果没有驱动无法连接数据库，更无法操作数据库。

JPA 即 Java Persistence API，用于对象持久化的 API 是一个规范，Java EE 5.0 平台标准的 ORM 规范使程序以统一的方式访问持久化层。Hibernate，TopLink 和 OpenJPA 等都是 JPA 的一种实现。从功能上来说 JPA 是 Hibernate 功能的一个子集。JPA 有以下的优势：

1. 标准化，提供相同的 API，保证了基于 JPA 开发的企业应用能够经过少量的修改就能够在同步的 JPA 框架下运行。
2. 简单易用，集成方便；JPA 主要的目标之一就提供更加简单地编程模型，在 JPA 框架下创建实体和创建 Java 类一样简单。
3. 查询能力强；JPA 的查询语言是面向对象的， JPA 定义了独特的 JPQL，能够支持批量更新和修改，JOIN，Group By 等高级特性。
4. 支持面向对象的高级特性；JPA 能够支持例如继承，多态和类之间的复杂关系，最大限度的使用面向对象的模型。

JPA 包括以下 3 个方面的技术：

1. ORM 映射元数据；JPA 支持 XML 和 注解两种元数据形势，元数据描述对象和表之间的映射关系，框架也是根据此将实体对象持久化到数据库中。
2. JPA 的 API；用来操作实体对象，执行 CRUD 操作，框架在后台完成的所有事情，开发者可以不用操作繁琐的 JDBC 和 SQL 代码。
3. 查询语言；是持久化操作中一个很重要的方便，通过面向对象而非面向数据库的查询数据，避免程序和具体的 SQL 的高度耦合。

使用 JPA 进行持久化对象的步骤主要如下：

1. 创建 `persistence.xml` 在这个文件中配置持久化信息；例如执行需要连接的数据库，指定 JPA 使用哪个持久化框架，使用到框架的基本配置属性。
2. 创建实体类；使用注解来描述实体类和数据库表之间的映射关系。
3. 使用 JPA API 完成数据增删改查操作；即创建 `EntityManagerFactory`  和 `EntityManager`。

一个简单的基于 Hibernate 的 JPA 项目如下：

首先是 `pom.xml` 添加相关的依赖：

```xml
<dependencies>
  <!-- https://mvnrepository.com/artifact/org.hibernate/hibernate-core -->
  <dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.4.10.Final</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.18</version>
  </dependency>
</dependencies>
```

然后再 resource 目录下添加 `/META-INF/persistence.xml` 文件，配置持久化框架的相关信息：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence" version="2.0">
    <persistence-unit name="NewPersistenceUnit">
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
        <class>com.xinyue.myjpa.model.Book</class>
        <properties>
            <property name="hibernate.connection.url" value="jdbc:mysql:///xinyue"/>
            <property name="hibernate.connection.driver_class" value="com.mysql.jdbc.Driver"/>
            <property name="hibernate.connection.username" value="root"/>
            <property name="hibernate.connection.password" value="123456"/>
            <property name="hibernate.archive.autodetection" value="class"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.hbm2ddl.auto" value="update"/>
        </properties>
    </persistence-unit>
</persistence>
```

然后创建一个简单地 实体类：

```java
@Entity(name = "jpa_book")
public class Book {
    private Integer id;
    private String name;
    private Double price;
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Double getPrice() {
        return price;
    }
    public void setPrice(Double price) {
        this.price = price;
    }
}
```

最后编写 Main 类测试

```java
public class MyJpa {
    public static void main(String[] args) {
        EntityManagerFactory factory = Persistence.createEntityManagerFactory("NewPersistenceUnit");
        EntityManager manager = factory.createEntityManager();
        EntityTransaction transaction = manager.getTransaction();

        transaction.begin();

        Book book = new Book();
        book.setName("Test Book");
        book.setPrice(12.5d);

        manager.persist(book);

        transaction.commit();

        manager.close();
        factory.close();
    }
}
```

最后打开数据库查看数据已经存储到表中了：

![数据库结果](http://img.programya.com/Snipaste_2019-12-21_10-48-28.png)

在 Book 这个实体类中可以看到几个相关的注解：

首先是 `@Entity` 表明这是一个实体类，如果类名称和表名称不一致，可以通过设置 name 属性来修改该类对应的表名称。

然后是 `@Id` 标注用于声明一个实体类的属性映射为数据库的主键列，可以放在 `getter` 方法上。

在制定了主键之后需要说明生成主键的策略，使用 `@GeneratedValue` 方式标注数据生成的方式，通过 `strategy` 属性制定生成的策略，主要有以下几种：

1. `IDENTITY`  采用数据库 ID 自增长的方案。
2. `AUTO` 默认选项，JPA 自动生成合适的策略，Oracle 不支持这种方式。
3. `SEQUENCE` 通过序列生成主键，通过 `@SequenceGenerator` 注解制定序列名称，MySQL 不支持这种方式。
4. `TABLE` 通过表产生主键，框架借由表模拟序列产生注解，使用该策略时数据库移植更为方便。

然后是 `@Basic` 是一个默认注解，表示一个属性到数据库表的字段映射，对于没有任何标注的 getter 方法默认即为 `@Basic` 注解。其有 `fetch` 和 `optional` 两个属性，其中 `fetch` 表示该属性的读取策略有两种选择 `FetchType.EAGER` 和 `FetchType.LAZY` 前者表示及时获取后者表示延时加载。默认是 `EAGER`。`optional`注解表示该属性是否可以为 `null` 默认是  `true`。

然后是 `@Column` 可以通过其 `name` 属性将类属性 mapping 到数据库字段上。也可以设置 `unique`， `nullable`，`length` 等属性。此外对于 Java 数据类型和 数据库数据类型不一致的情况可以使用 `columnDefinition` 属性进行指定，例如指定 String 类型字段为 varchar 或者 text，`columnDefinition = "text"`。

然后是 `@Transient` 表示该属性并非一个数据库表字段，ORM 框架忽略该属性。

然后是 `@Temporal` 用于区分设置日期和时间的。例如 `TemporalType.TIMESTAMP`， `TemporalType.DATE` 等。

然后是 `@TableGenerator` 注解，如果主键生成策略时 Table `@GeneratedValue(strategy = GenerationType.TABLE, generator = "ID_GENERATOR")` 那么需要设置对应的 Table Generator。

```java
@TableGenerator(
  // 生成器名称
  name = "ID_GENERATOR",
  // 对应的表
  table = "JPA_ID_GENERATOR",
  // 每次增长多少
  allocationSize = 1,
  // 初始值是多少
  initialValue = 1,
  // 存储序列的列
  pkColumnName = "PK_NAME",
  // 存储序列的 value
  pkColumnValue = "BOOK",
  // 当前 ID 值的列
  valueColumnName = "ID_VAL"
)
```

