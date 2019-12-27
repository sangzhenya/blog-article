## 动态 SQL

使用 MyBatis 可以很轻松的完成动态 SQL 的拼写。

一个简单的使用 `if ` 拼接的查询方法如下：

```xml
<!-- List<User> queryUser(@Param("id") Integer id, @Param("name") String name, @Param("firstName") String firstName, @Param("lastName") String lastName); -->
<select id="queryUser" resultType="User" databaseId="mysql">
  select * from user
  where 1 = 1
  <if test="id != null">and  id = #{id} </if>
  <if test="name != null and name.trim() != ''">and  name = #{name} </if>
  <if test="firstName != null and firstName != ''">and  first_name = #{firstName} </if>
  <if test="lastName != null and lastName != ''">and  last_name = #{lastName} </if>
</select>
```

当然也可以使用 `where` 标签去省略掉 `where` 关键字：

```xml
 <select id="queryUser" resultType="User" databaseId="mysql">
   select * from user
   <where>
     <if test="id != null">id = #{id} </if>
     <if test="name != null and name.trim() != ''">and  name = #{name} </if>
     <if test="firstName != null and firstName != ''">and  first_name = #{firstName} </if>
     <if test="lastName != null and lastName != ''">and  last_name = #{lastName} </if>
   </where>
</select>
```

也可以使用 `trim` 标签：

```xml
<!-- trim: prefix 为 SQL 添加的前缀，prefixOverrides 从 SQL 前端去掉的内容
 suffix 为 SQL 添加的后缀，suffixOverrides 从 SQK 末端去掉的内容-->
<select id="queryUser" resultType="User" databaseId="mysql">
  select * from user 
  <trim prefix="where" prefixOverrides="and">
    <if test="id != null">and id = #{id} </if>
    <if test="name != null and name.trim() != ''">and  name = #{name} </if>
    <if test="firstName != null and firstName != ''">and  first_name = #{firstName} </if>
    <if test="lastName != null and lastName != ''">and  last_name = #{lastName} </if>
  </trim>
</select>
```

`choose` 标签可以选择不同的分支，和 java 中的 `switch` 类似

```xml
<select id="queryUserBySingleCriteria" resultType="User" databaseId="mysql">
  select * from user
  <where>
    <choose>
      <when test="id != null">id = #{id}</when>
      <when test="name != null">name = #{name}</when>
      <when test="firstName != null">first_name = #{firstName}</when>
      <when test="lastName != null">last_name = #{lastName}</when>
    </choose>
  </where>
</select>
```

`set` 标签和 `where` 标签相似，为 SQL 添加一个 `set` 关键字

```xml
<select id="updateUserByCriteria" >
  update user
  <set>
    <if test="id != null">id = #{id}, </if>
    <if test="name != null and name.trim() != ''">name = #{name}, </if>
    <if test="firstName != null and firstName != ''">first_name = #{firstName}, </if>
    <if test="lastName != null and lastName != ''">last_name = #{lastName} </if>
  </set>
  where id = #{id}
</select>
<!-- 当然也可以使用 trim 标签实现，只需要在 prefix='set' suffixOverrides=',' 即可  -->
```

`foreach` 可以用来遍历集合如下：

```xml
<!-- collection 要遍历的集合；item 遍历过程中使用的变量；open 结果前拼接的字符；
close 结果后拼接的字符；separator 分隔符；index 索引（Map 的时候对应的是 key） -->
<select id="findUsersByIds" resultType="User">
  select * from user
  <foreach collection="ids" item="user_id" open="where id in (" close=")" separator=",">
    #{user_id},
  </foreach>
</select>
```

SQL 的抽取和批量更新简单的例子如下：

```xml
<!-- 公用 SQL 片段抽取 -->
<sql id="commonColumns">
  <!-- 判断是什么数据库 -->
  <if test="_databaseId=='oracle'">
    user_id,name, `password`
  </if>
  <if test="_databaseId=='mysql'">
    id, name, `password`
  </if>
</sql>

<!-- 批量插入 -->
<insert id="batchCreateUser"  databaseId="mysql">
  insert into user (
  <!-- include 也可以自定义 property 例如 
		<property name="testProperty" value="demo"/> 在 sql 标签中可以使用 ${testProperty} 获取值
	-->
  <include refid="commonColumns" />
  ) values
  <foreach collection="users" item="user" separator=",">
    (#{user.name}, #{user.password})
  </foreach>
</insert>
<!-- 当然也可以通过 foreach 生成多条 Insert SQL 的实现同样的效果；
不过这样需要在连接 DB 的 url 配置 allowMultiQueries=true -->

<!-- 对于 Oracle 数据库而言不支持这样的方式，所以可以使用以下的两种方式 -->
<!-- 方式 1-->
<insert id="batchCreateUser" databaseId="oracle">
  <foreach collection="users" item="user" open="begin" close="end;">
    insert into user(id, user_name, user_password) 
    values(user_seq.nextval, #{user.name},#{user.password});
  </foreach> 
</insert>

<!-- 方式 2-->
<insert id="batchCreateUser" databaseId="oracle">
  insert into employees(
  <include refid="commonColumns" />
  )
  <foreach collection="users" item="user" separator="union"
           open="select user_seq.nextval, user_name, user_password from("
           close=")">
    select #{user.name} user_name, #{user.password} user_password from dual
  </foreach>
</insert>
```

此外 MyBatis 还内置了两个参数分别为 `_parameter` 和 `_databaseId`。

其中 `_parameter` 代表整个参数。如果是单个参数那么就是这个参数；如果是多个参数会封装成一个 map，其就是这个 map。

其中 `_databaseId` 如果配置了 `databaseIdProvider` 其就是当前数据库的别名。

此外 MyBatis 还提供了 bind 函数将 ONGL 的表达式的结果绑定到一个变量上，方便后来引用这个变量。简单的例如如下：

```xml
<select id="testParameterAndBind" resultType="User">
  select * from user
  <bind name="_name" value="'%'+name+'%'"/>
  <if test="_databaseId=='mysql'">
    select * from user
    <if test="_parameter!=null">
      where name like #{_name}
    </if>
  </if>
</select>
```

