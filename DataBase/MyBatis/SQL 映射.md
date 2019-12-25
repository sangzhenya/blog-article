## SQL 映射

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

获取自增主键值

Oracle 等使用 查询获取主键值（selectKey）

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

