## MyBatis SQL 映射

一个包含增删改查的 MyBatis Mapper 文件如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.xinyue.imybatis.mapper.UserMapper">
    <select id="getUserById" resultType="User" databaseId="mysql">
		select id, name, password, first_name firstName, last_name lastName from user where id = #{id}
	</select>
    <insert id="createUser" parameterType="User">
        insert  into user (name, password, first_name, last_name) values (#{name}, #{password}, #{firstName}, #{lastName})
    </insert>
    <update id="updateUser">
        update user set name = #{name}, password=#{password}, first_name=#{firstName} where id=#{id}
    </update>
    <delete id="deleteUser">
        delete from user where id = #{id}
    </delete>
</mapper>
```

一个简单的测试样例如下：

```java
@Test
public void testCreate() {
  try {
    SqlSessionFactory sqlSessionFactory = getSqlSessionFactory();
    try (SqlSession openSession = sqlSessionFactory.openSession()) {
      UserMapper mapper = openSession.getMapper(UserMapper.class);
      User user = new User();
      user.setName("xinyue3");
      user.setPassword("password3");
      mapper.createUser(user);
      System.out.println(user);
      openSession.commit();
    }
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

MyBaits 允许增删改直接定义以下类型的返回值：`Integer，Long，Boolean, void`。另外如果使用` sqlSessionFactory.openSession() `的方式获取 session 则需要手动 commit；如果使用 `sqlSessionFactory.openSession(true)` 则会自动提交。

### 主键

对于能自增主键的可以使用下面的方法获取主键：

```xml
<!--  获取自增主键的值：useGeneratedKeys 表示使用自增主键的值；keyProperty 表示将自增组件的值放到属性的名称上  -->
<insert id="createUser" parameterType="User" useGeneratedKeys="true" keyProperty="id">
  insert  into user (name, password, first_name, last_name) values (#{name}, #{password}, #{firstName}, #{lastName})
</insert>
```

对于 Oracle 等使用不能自增主键的则需要查询获取主键值，如下所示：

```xml
<insert id="createUser" databaseId="oracle">
    <!-- keyProperty: 查出的主键值封装给javaBean的哪个属性
        order 类型有以下两种：
        BEFORE运行顺序： 先运行selectKey查询id的sql；查出id值封装给javaBean的id属性，再运行插入的sql；就可以取出id属性对应的值
        AFTER运行顺序： 先运行插入的sql（从序列中取出新值作为id）；再运行selectKey查询id的sql；
        resultType:查出的数据的返回值类型 -->
    <selectKey keyProperty="id" order="BEFORE" resultType="Integer">
      <!-- 查询主键的sql语句 -->
      select EMPLOYEES_SEQ.nextval from dual
    </selectKey>
    insert  into user (id, name, password, first_name, last_name) values (#{id}, #{name}, #{password}, #{firstName}, #{lastName})
</insert>
<!--<insert id="createUser" databaseId="oracle">
        <selectKey keyProperty="id" order="BEFORE" resultType="Integer">
            select EMPLOYEES_SEQ.nextval from dual
        </selectKey>
        insert into employees(EMPLOYEE_ID,LAST_NAME,EMAIL)
        values(employees_seq.nextval,#{lastName},#{email})
</insert>-->
```



### MyBatis 对于参数的处理

#### 单个参数

没有特殊处理，使用 #{参数名/任意名称} 取出参数值。

#### 多个参数

多个参数会被封装成一个 Map， key 可以用 `param1, param2` 当然也可以用 `arg1, arg0`，value 就是传入的值。#{key} 就是从 map 中获取指定 key 的值。例如下面一个方法

```java
User getUserByFirstNameAndListName(String firstName, String lastName);
// Available parameters are [arg1, arg0, param1, param2]
```

如果使用了命名参数则可以使用命名参数的中值获取，例如：

```java
User getUserByFirstNameAndListName(@Param("firstName") String firstName, @Param("lastName") String lastName);
// Available parameters are [firstName, lastName, param1, param2]
```

如果参数是我们实体对象或者是 Map，则分别可以通过属性名和属性的 Key 获取对应的 Value，`#{key}`。

如果参数是 Collection 或者数组，也会封装到 Map 中，`Collection -> collection; List -> list; 数组 -> array`。例如一个 ` List<Integer> ids ` 获取第 0 个元素则可以使用 `#{list[0]}` 获取。

下面看一下解析参数的流程，主要是 `ParamNameResolver` 方法：

```java
// ParamNameResolver
public ParamNameResolver(Configuration config, Method method) {
  // 获取方法参数列表类型数组
  final Class<?>[] paramTypes = method.getParameterTypes();
  // 获取方法参数注解数组
  final Annotation[][] paramAnnotations = method.getParameterAnnotations();
  // 存放参数名称
  final SortedMap<Integer, String> map = new TreeMap<>();
  // 获取参数数量
  int paramCount = paramAnnotations.length;
  // 遍历参数放到参数名称 Map 中
  for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
    if (isSpecialParameter(paramTypes[paramIndex])) {
      // 剔除特殊的参数类型：RowBounds.class.isAssignableFrom(clazz) || ResultHandler.class.isAssignableFrom(clazz);
      continue;
    }
    String name = null;
    // 从 @Param 参数注解中获取注解名称作为 name
    for (Annotation annotation : paramAnnotations[paramIndex]) {
      if (annotation instanceof Param) {
        hasParamAnnotation = true;
        name = ((Param) annotation).value();
        break;
      }
    }
    // 如果找到即没有使用 @Param 注解
    if (name == null) {
      // 如果配置了获取 useActualParamName 为 true，默认为 true
      if (config.isUseActualParamName()) {
        // 获取真实的参数名称
        // 编译使用 JDK 8 且带有 -parameters 注解的时候才可以获取真实的参数名称作为 name，否则返回 arg0, arg1 这样的值作为 name
        name = getActualParamName(method, paramIndex);
      }
      if (name == null) {
        // 如果还是没有找到则使用 index 作为 name ("0", "1", ...)
        name = String.valueOf(map.size());
      }
    }
    // 放到参数 name map中
    map.put(paramIndex, name);
  }
  // 放到类属性上
  names = Collections.unmodifiableSortedMap(map);
}

public Object getNamedParams(Object[] args) {
  // 获取参数的数量
  final int paramCount = names.size();
  if (args == null || paramCount == 0) {
    // 如果没有参数则直接返回 null
    return null;
  } else if (!hasParamAnnotation && paramCount == 1) {‘
    // 如果没有参数注解且只有一个参数则直接返回第一个参数
    return args[names.firstKey()];
  } else {
    // 新建一个参数值 Map
    final Map<String, Object> param = new ParamMap<>();
    int i = 0;
    for (Map.Entry<Integer, String> entry : names.entrySet()) {
      // 把参数名称和参数值放到 Map 中
      param.put(entry.getValue(), args[entry.getKey()]);
      // 添加通用参数名称例如 (param1, param2, ...)
      final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
      // 确保通用的参数不会覆盖 @Param 中定义的属性
      if (!names.containsValue(genericParamName)) {
        param.put(genericParamName, args[entry.getKey()]);
      }
      i++;
    }
    // 返回参数值 Map
    return param;
  }
}
```

### `#` 和 `$`

`#{}` 和 `${}` 均可以从 map 中获取 pojo 对象属性的值。其区别如下：

对于 `#{}` 是以预编译的形式直接将参数设置到 SQL 语句中，使用 `PreparableStatement` 防止 SQL 注入。其可以规定参数的一些规则如：`javaType`, `jdbcType`, `mode`, `numericScale`, `resultMap`, `typeHanlder`, `jdbcTypeName`。其中 `jdbcType` 在某些特殊的情况下需要设置，例如如果数据为 null，MyBatis默认将 null 映射成了 JDBC 原生的 OTHER 类型，所以有些数据库不能识别 MyBatis 对 null 的处理。则可以设置如下 `#{name,jdbcType=NULL}`。当然也可以在全局配置 `<setting name="jdbcTypeForNull" value="NULL"/>`。例如：

```mysql
#{age,javaType=int,jdbcType=NUMERIC,typeHandler=MyTypeHandler}
#{height,javaType=double,jdbcType=NUMERIC,numericScale=2}
#{department, mode=OUT, jdbcType=CURSOR, javaType=ResultSet, resultMap=departmentResultMap}
```

对于 `${}` 取出值直接拼接到 SQL 语句中，可能会有安全问题。但是对于一些 JDBC 不能使用占位符的情况可以使用其取值。例如：`ORDER BY ${columnName}`, `select * from ${year}_salary where xxx;`，简单可以认作为字符串替换。

### Select

select 主要属性如下：

```properties
parameterType=可选属性，传入这条语句的限定名或别名。MyBatis 可以通过 TypeHandler 判断出具体传入数据的参数。
resultType=返回的期望的类的完全限定名或别名，如果是集合或者 Map 则设置包含的类型
resultMap=外部 resultMap 的命名引用
flushCache=任何使用只要语句被调用都会清空本地和二级缓存，默认为 false
useCache=使用耳机缓存，对于 select 默认为 true
timeout=超时时间
fetchSize=每次批量返回的结果行数
statementType=STATEMENT，PERPARED 或 CALLABLE 的一个
resultSetType=FORWARD_ONLY,SCROLL_SENSITIVE 或 SCROLL_INSENSITIVE 中一个
databaseId=指定特定数据库的语句
resultOrdered=针对嵌套结果的 select 语句使用，例如包含嵌套结果或是分组，返回一个主结果行，就不会出现对前面结果集引用的情况，避免嵌套获取结果集的时候内存不够用的情况
resultSets=对多结果集的情况适用，将列出语句执行后返回的结果集并给每个结果集一个名称。
```



如果查询需要返回一个 List，则需要将 `select` 中的 `resultType` 设置为 List 内的元素。

```xml
<!-- List<User> findByNameLike(String name); -->
<select id="findByNameLike" resultType="User">
  select id, name, password, first_name firstName, last_name lastName from user where name like #{name}
</select>
```

如果查询要返回 Map 可以设置如下：

```xml
<!-- Map<String, Object> getMapById(Integer id); -->
<select id="getMapById" resultType="map" databaseId="mysql">
  select id, name, password, first_name firstName, last_name lastName from user where id = #{id}
</select>
```

```xml
<!--
// 设置 Map 的 Key
@MapKey("id")
Map<Integer, User> getUserMap(String name);
-->
<select id="getUserMap" resultType="User">
  select * from user where name like #{name}
</select>
```

可以使用 `resultMap` 自定义封装规则：

```xml
<!-- 使用级联关系 -->
<!-- User getCustomizedUserById(Integer id); -->
<resultMap id="CustomizedUser" type="com.xinyue.imybatis.model.User">
   <id column="id" property="id" />
   <result column="name" property="name" />
   <result column="first_name" property="firstName" />
   <result column="last_name" property="lastName" />
   <!--   未列出的列也会进行自动封装     -->
</resultMap>

<select id="getCustomizedUserById" resultMap="CustomizedUser" databaseId="mysql">
  select id, name, password, first_name firstName, last_name lastName from user where id = #{id}
</select>
```

```xml
<!-- 使用 association 属性 -->
<!--User getUserWithZone(Integer id); -->
<resultMap id="UserWithZone" type="com.xinyue.imybatis.model.User">
  <id column="id" property="id" />
  <result column="name" property="name" />
  <result column="first_name" property="firstName" />
  <result column="last_name" property="lastName" />
  <!-- <result column="z_id" property="zone.id" />
  <result column="z_name" property="zone.name" /> -->
  <association property="zone" javaType="com.xinyue.imybatis.model.Zone">
    <result column="z_id" property="id" />
    <result column="z_name" property="name" />
  </association>
</resultMap>
<select id="getUserWithZone" resultMap="UserWithZone">
  select u.id, u.name, u.`password`, u.first_name, u.last_name, z.id z_id, z.name as z_name from user u, zone z where u.zone_id = z.id and u.id = #{id};
</select>
```

```xml
<!-- 使用 association 分布查询 -->
<!-- User getUserWithZoneSeq(Integer id); -->
<resultMap id="UserWithZoneSeq" type="com.xinyue.imybatis.model.User">
  <id column="id" property="id" />
  <result column="name" property="name" />
  <association property="zone" select="com.xinyue.imybatis.mapper.ZoneMapper.getById" column="zone_id">
  </association>
</resultMap>
<select id="getUserWithZoneSeq" resultMap="UserWithZoneSeq">
  select * from user u where u.id = #{id};
</select>
```

如果需要延迟加载需要在 `setting` 中配置如下：

```xml
<setting name="lazyLoadingEnabled" value="true" />
<setting name="aggressiveLazyLoading" value="false" />
```

对于查询结果中有集合的情况如下：

```xml
<!-- Zone getZoneWithUsersById(Integer id); -->
<resultMap id="ZoneWithUsers" type="Zone">
  <id column="id" property="id" />
  <result column="name" property="name" />
  <collection property="userList" ofType="User">
    <id column="u_id" property="id" />
    <result column="u_name" property="name" />
    <result column="u_password" property="password" />
  </collection>
</resultMap>
<select id="getZoneWithUsersById" resultMap="ZoneWithUsers">
  select ze.id, ze.name, ze.zone_code, ur.id u_id, ur.name u_name, ur.`password` u_password  from zone ze, user ur where ze.id = ur.zone_id and ze.id = #{id};
</select>
```

```xml
<!-- 延迟查询 -->
<!-- Zone getZoneWithUsersByIdSeq(Integer id); -->
<resultMap id="ZoneWithUsersSeq" type="Zone">
  <id column="id" property="id"/>
  <result column="name" property="name"/>
  <collection property="userList" select="com.xinyue.imybatis.mapper.UserMapper.getUsersByZoneId" column="id">
  </collection>
</resultMap>
<select id="getZoneWithUsersByIdSeq" resultMap="ZoneWithUsersSeq">
  select * from zone where id = #{id}
</select>
```

如果想要想 association 或 collection 传递多个值，则可以将多列的值封装成 Map，如下

`column="{key1=column1, key2=column2}"`。

如果想要根据查询结果在进行不同的查询，则可以使用 `discriminator` 一个简单的样例如下：

```xml
<!-- User getUserWithZoneSeq(Integer id); -->
<resultMap type="User" id="CustomizedUserDis">
  <id column="id" property="id"/>
  <result column="name" property="name"/>
  <!-- column：指定判定的列名; javaType：列值对应的java类型  -->
  <discriminator javaType="string" column="first_name">
    <case value="Lin" resultType="User">
      <association property="zone"  select="com.xinyue.imybatis.mapper.ZoneMapper.getById" column="zone_id">
      </association>
    </case>
    <case value="Xin" resultType="User">
      <id column="id" property="id"/>
      <result column="first_name" property="lastName"/>
    </case>
  </discriminator>
</resultMap>
<select id="getUserByIdDis" resultMap="CustomizedUserDis" databaseId="mysql">
  select * from user where id = #{id}
</select>
```



