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

