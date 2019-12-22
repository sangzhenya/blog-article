## JPQL 

JPQL  语言，即 Java Persistence Query Language 是一种和 SQL 非常相似的中间性和对象查询语言，其最终会被编译成针对于不同底层数据库的 SQL 查询，从而屏蔽掉不同数据库的差异。JPQL 的语句都是通过 Query 接口封装执行的。

Query 接口主要有以下几个方法：

1. `int executeUpdate();`  执行 Update 或者 Delete 语句。
2. `List getResultList();` 执行 Select 语句并返回结果实体列表。
3. `Object getSingleResult();` 用于执行只返回单个结果实体的 Select 语句。
4. `Query setFirstResult(int startPosition);` 用于设置从哪个实体记录开始返回查询结果。
5. `Query setMaxResults(int maxResult);` 用于设置返回结果实体的最大数。
6. `Query setFlushMode(FlushModeType flushMode);` 设置查询对象的 Flush 模式，AUTO 或者 COMMIT。
7. `Query setHint(String hintName, Object value);` 设置与查询对象相关的特定供应商参数或提示信息。
8. `Query setParameter(int position, Object value);` 设置查询语句指定位置的参数赋值，position 指定参数序号，value 为赋值给参数的值。
9. `Query setParameter(int position, Date value, TemporalType temporalType);` 为查询语句设置 Date 值，position 为参数序号，value 为赋值给参数的值，temporalType 是枚举 DATE，TIME 已经 TIMESTAMP，用于将 Java 的 Date 型树脂转换为数据库支持的日期类型。
10. `Query setParameter(int position, Calendar value, TemporalType temporalType);` 同上。



JPA 同样可以设置二级缓存，在 `pom.xml` 中添加依赖如下：

```xml
<!-- https://mvnrepository.com/artifact/org.hibernate/hibernate-ehcache -->
<dependency>
  <groupId>org.hibernate</groupId>
  <artifactId>hibernate-ehcache</artifactId>
  <version>5.4.10.Final</version>
</dependency>
```

在 `persistence-unit` 中添加属性：

```xml
 <!--
            配置耳机缓存策略：
            ALL: 所有实体类都被缓存
            NONE: 所有实体类都不会被缓存
            ENABLE_SELECTIVE：标识了 @Cacheable(true) 被缓存
            DISABLE_SELECTIVE: 除了标识  @Cacheable(true) 被缓存
            UNSPECIFIED: 默认值
-->
<shared-cache-mode>UNSPECIFIED</shared-cache-mode>

<properties>
  <property name="hibernate.connection.url" value="jdbc:mysql:///xinyue"/>
  <property name="hibernate.connection.driver_class" value="com.mysql.jdbc.Driver"/>
  <property name="hibernate.connection.username" value="root"/>
  <property name="hibernate.connection.password" value="36003981D"/>
  <property name="hibernate.archive.autodetection" value="class"/>
  <property name="hibernate.show_sql" value="true"/>
  <property name="hibernate.format_sql" value="true"/>
  <property name="hibernate.hbm2ddl.auto" value="update"/>

  <property name="hibernate.cache.use_second_level_cache" value="true"/>
  <property name="hibernate.cache.region.factory_class" value="org.hibernate.cache.ehcache.internal.EhcacheRegionFactory"/>
  <property name="hibernate.cache.use_query_cache" value="true"/>
</properties>
```

Select 语句用来执行查询，其语法可以表示为：`SELECT FROM [WHERE] [GROUP BY] [HAVING] [ORDER BY]` 的形式。

```java
// 一些常见的查询如下
// 简单的设置参数查询并使用 Order By 排序
Query query = manager.createQuery("FROM student s where s.id > ?1 order by s.id")
                .setParameter(1, 11);
List<Student> students = query.getResultList();
System.out.println(students.size());

// 查询部分字段并封装成一个对象， Student 中需要有相应的构造函数
Student student = (Student) manager.createQuery("SELECT new student(st.id, st.name, st.age) " +
                "FROM student st where st.id = ?1").setParameter(1, 11).getSingleResult();
System.out.println(student);

public Student(Integer id, String name, Integer age) {
  this.id = id;
  this.name = name;
  this.age = age;
}

// 使用 NamedQuery 查询， 在实体类上需要现金 namedQuery 注解
Student student = (Student) manager.createNamedQuery("selectStudentById")
                .setParameter(1, 11).getSingleResult();
System.out.println(student);

@NamedQuery(name = "selectStudentById", query = "FROM student s where s.id = ?1")
@Entity(name = "student")
public class Student {}

// 使用 Native 查询
Object single = manager.createNativeQuery("select age from student where id = ?1").setParameter(1, 11).getSingleResult();
System.out.println(single);

// 使用缓存
Query query = manager.createQuery("FROM student s where s.id > ?1")
                .setParameter(1, 11)
                .setHint(QueryHints.HINT_CACHEABLE, true);
List<Student> students = query.getResultList();
System.out.println(students.size());

query = manager.createQuery("FROM student s where s.id > ?1")
  .setParameter(1, 11)
  .setHint(QueryHints.HINT_CACHEABLE, true);
students = query.getResultList();
System.out.println(students.size());

```

同样也可以使用 `GROUP BY HAVING`，关联查询和子查询。

JPQL 也提供了一些内建函数。

字符函数：`concat`, `substring`, `trim`, `lower`, `upper`, `length`, `locate`(从第一个参数中查找第二个参数出现的位置，查不到返回 0)。

算数函数：`abs`, `mod`, `sqrt`, `size`。

日期函数：`current_date`, `current_time`, `current_timestamp`。



Update 函数和 Delete 函数简单例子如下：

```java
// Update 语句
Query updateQuery = manager.createQuery("UPDATE student st set st.age = ?1 where st.id > ?2");
// Delete 语句
Query deleteQuery = manager.createQuery("delete from student st where st.id < ?1");
```

