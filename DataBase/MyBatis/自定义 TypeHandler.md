## MyBaits 自定义 TypeHandler

MyBaits 支持通过自定义的 TypeHandler 的形式来设置参数或从结果中取出数据集的时候的封装策略。主要有以下几步：

1. 实现 `TypeHanlder`结果或者继承 `BaseTypeHandler`
2. 使用 `@MappedTypes` 定义处理的 Java 类型，使用 `@MapJdbcTypes` 定义 jdbc type 类型
3. 在参数处理或自定义结果集处理的时候标明使用自定义的 TypeHandler 处理。也可以在全局配置 TypeHandler 要处理的 javaType

定义一个枚举类放到 User 上：

```java
public enum UserStatus {
    ACTIVE(1, "User Active"), INACTIVE(-1, "User InActive");

    private Integer code;
    private String desc;
    UserStatus(Integer code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getDesc() {
        return desc;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }

    public static UserStatus getByCode(Integer code) {
        switch (code) {
            case 1:
                return UserStatus.ACTIVE;
            case -1:
                return UserStatus.INACTIVE;
        }
        return null;
    }
}
```

一个自定义的枚举类型的处理器如下：

```java
public class MyUserStatusHandler implements TypeHandler<UserStatus> {
    @Override
    public void setParameter(PreparedStatement ps, int i, UserStatus parameter, JdbcType jdbcType) throws SQLException {
        ps.setInt(i, parameter.getCode());
    }

    @Override
    public UserStatus getResult(ResultSet rs, String columnName) throws SQLException {
        int code = rs.getInt(columnName);
        return UserStatus.getByCode(code);
    }

    @Override
    public UserStatus getResult(ResultSet rs, int columnIndex) throws SQLException {
        int code = rs.getInt(columnIndex);
        return UserStatus.getByCode(code);
    }

    @Override
    public UserStatus getResult(CallableStatement cs, int columnIndex) throws SQLException {
        int code = cs.getInt(columnIndex);
        return UserStatus.getByCode(code);
    }

}
```

最后在全局配置中注册该处理类：

```xml
<typeHandlers>
  <typeHandler handler="com.xinyue.imybatis.handler.MyUserStatusHandler" javaType="com.xinyue.imybatis.enmu.UserStatus" />
</typeHandlers>
```

以上就完成了一个简单地处理枚举类型自定义类了。