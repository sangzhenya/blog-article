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

![数据库结果](http://img.sangzhenya.com/Snipaste_2019-12-21_10-48-28.png)

