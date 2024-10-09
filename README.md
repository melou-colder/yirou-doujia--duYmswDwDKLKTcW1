[合集 \- Java后端(15\)](https://github.com)[1\.【解决方案】如何使用 Http API 代替 OpenFeign 进行远程服务调用01\-22](https://github.com/CodeBlogMan/p/17979942)[2\.【深度思考】一线开发大头兵对于工作的感悟分享01\-29](https://github.com/CodeBlogMan/p/17983370)[3\.【进阶篇】Java 实际开发中积累的几个小技巧（一）02\-04](https://github.com/CodeBlogMan/p/18005657)[4\.【设计模式】分享 Java 开发中常用到的设计模式（一）2023\-08\-09](https://github.com/CodeBlogMan/p/17594211.html)[5\.【解决方案】Java 互联网项目如何防止集合堆内存溢出（一）02\-20](https://github.com/CodeBlogMan/p/18022444)[6\.【主流技术】日常工作中关于 JSON 转换的经验大全（Java）03\-11](https://github.com/CodeBlogMan/p/18022825)[7\.【解决方案】Spring动态定时任务之ScheduledTaskRegistrar2023\-01\-31](https://github.com/CodeBlogMan/p/17079955.html)[8\.【主流技术】聊一聊对 Mybatis Plus 的理解与应用2022\-06\-13](https://github.com/CodeBlogMan/p/16369730.html)[9\.【Java 进阶】详细探究 Spring 框架中的注解与反射2022\-06\-08](https://github.com/CodeBlogMan/p/16354607.html)[10\.【进阶篇】使用 Stream 流和 Lambda 组装复杂父子树形结构01\-15](https://github.com/CodeBlogMan/p/17965824)[11\.【进阶篇】Java 实际开发中积累的几个小技巧（二）04\-16](https://github.com/CodeBlogMan/p/18135597):[veee加速器](https://liuyunzhuge.com)[12\.【进阶篇】使用 Stream 流对比两个集合的常用操作分享05\-27](https://github.com/CodeBlogMan/p/18156081)[13\.【进阶篇】Java 项目中对使用递归的理解分享07\-02](https://github.com/CodeBlogMan/p/18180395)[14\.【进阶篇】一文搞清楚网页发起 HTTP 请求调用的完整过程07\-15](https://github.com/CodeBlogMan/p/18249663)15\.【解决方案】基于数据库驱动的自定义 TypeHandler 处理器10\-09收起
目录* [前言](https://github.com)
* [一、TypeHandler 简介](https://github.com)
	+ [1\.1转换步骤](https://github.com)
	+ [1\.2转换规则](https://github.com)
* [二、JSON 转换](https://github.com)
* [三、枚举转换](https://github.com)
* [四、文章小结](https://github.com)

### 前言


笔者在最近的项目开发中，频繁地遇到了 Java 类型与 JDBC 类型之间的2个转换问题：


* 数据库的 varchar 类型字段，需要存储 Java 实体中的 JSON 字符串
* 数据库的 int 类型字段，需要存储 Java 实体中的 Enum 枚举


其实要处理也不麻烦，可以在每次入库地方的手动将 Java Bean 调用 JSON.*toJSONString()* 即可，取出数据库数据的时候再 JSON.*parseObject()*解析。再说处理枚举类型也并不难，无非就是手动将枚举的 int 型属性取出后 set 到数据库的int中去。


而本文要介绍的自定义 TypeHandler 处理器的作用，就是**自动处理 Java Bean 与数据库类型的转换**，提高编码效率，通过全局的统一处理省去繁琐的手动转换。




---


### 一、TypeHandler 简介


如果我们使用的是 Mybatis 或者是 Mybatis Plus 的话，在 SQL 语句执行过程中，无论是设置参数还是获取结果集，都需要通过 TypeHandler 进行类型转换。


MyBatis 提供了丰富的内置 TypeHandler 实现，以支持常见的数据类型转换，如以下几种：


表1\-1

![](https://img2024.cnblogs.com/blog/2458865/202408/2458865-20240811221532294-1913350796.png)

#### 1\.1转换步骤


当 MyBatis 执行一个预编译的 SQL 语句（如 INSERT、UPDATE 等）时，它需要将 Java 对象中的属性值设置到 SQL 语句中对应的占位符上。这个过程就是通过TypeHandler 来实现的。


具体步骤如下：


* MyBatis 会根据映射配置找到对应的 TypeHandle r实例，这个映射配置可以在 MyBatis 的配置文件或者 Mapper 的 XML 文件中定义；
* TypeHandler 实例会接收到 Java 对象中的属性值，并将其转换为 JDBC 能够识别的类型，这个转换过程是根据两者之间的映射关系来实现的；
* 转换后的值会被设置到 PreparedStatement 对象中对应的占位符上，以便数据库能够正确解析和执行 SQL 语句。


#### 1\.2转换规则


再次强调，TypeHandler 的核心功能是实现 Java 类型和 JDBC 类型之间的映射和转换，这个映射和转换规则是根据 Java 类型和 JDBC 类型的特性和语义来定义的。


* 对于基本数据类型（如 int、long、float等），MyBatis 提供了内置的 TypeHandler 实现，这些实现能够直接将 Java 基本数据类型转换为对应的 JDBC 基本数据类型，反之亦然。
* 对于复杂数据类型（如自定义对象、集合等），MyBatis 允许开发者自定义 TypeHandler 来实现复杂的类型转换逻辑。例如，开发者可以定义一个自定义的TypeHandler 来将数据库中的 JSON 字符串转换为 Java 中的对象，或者将 Java 对象转换为 JSON 字符串存储到数据库中。


下面两章就举两个例子来加以说明。




---


### 二、JSON 转换


应用的 .yml 配置文件新增以下：



```
mybatis-plus:
  type-handlers-package: #自定义 handler 类所在的包路径

```


```
/**
 * 作用：即 Java 实体属性可以直接使用 JSONObject 映射数据库的 varchar，方便入库、出库


 * 注意：需要在 .yml 配置文件上加上 {@code mybatis:type-handlers-package: 本类所在包路径}


 *
 * @param  该泛型即为需要转换成 varchar 的 Java 对象
 * @MappedTypes 注解很关键，指定了映射的类型
 */
@MappedTypes({JSONObject.class, JSONArray.class})
public class JSONTypeHandler  extends BaseTypeHandler {

    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, T param, JdbcType jdbcType) throws SQLException {
        //将指定的参数设置为给定的 Java String 值，数据库驱动程序及其转换成 varchar 类型
        preparedStatement.setString(i, JSON.toJSONString(param));

    }

    @Override
    public T getNullableResult(ResultSet resultSet, String columnName) throws SQLException {
        //这里根据字段名去拿到之前放进来的 jsonStr 值
        String jsonStr = resultSet.getString(columnName);
        return StringUtils.isNotBlank(jsonStr) ? JSON.parseObject(jsonStr, getRawType()) : null;
    }

    @Override
    public T getNullableResult(ResultSet resultSet, int columnIndex) throws SQLException {
        //这里是根据位置来确定字段，进而拿到该字段的值（之前放进来的 jsonStr）
        String jsonStr = resultSet.getString(columnIndex);
        return StringUtils.isNotBlank(jsonStr) ? JSON.parseObject(jsonStr, getRawType()) : null;
    }

    @Override
    public T getNullableResult(CallableStatement callableStatement, int columnIndex) throws SQLException {
        //这里是根据SQL存储过程里的字段位置来拿字段的值
        String jsonStr = callableStatement.getString(columnIndex);
        return StringUtils.isNotBlank(jsonStr) ? JSON.parseObject(jsonStr, getRawType()) : null;
    }

}

```



---


### 三、枚举转换



```
/**
 * 作用：将实体类中的枚举 code 映射为数据库的 int


 * 注意：需要在 .yml 配置文件上加上 {@code mybatis:type-handlers-package: 本类所在包路径}


 *
 * @param  该泛型即为需要处理的枚举对象，使用上界通配符来保证类型安全
 */
@MappedTypes(MyEnum.class)
public class EnumCodeTypeHandler extends MyEnum> extends BaseTypeHandler {

    private final Class type;

    /**
     * 记录枚举值和枚举的对应关系
     */
    private final Map enumMap = new ConcurrentHashMap<>();

    public EnumCodeTypeHandler(Class type) {
        Assert.notNull(type, "argument cannot be null");
        this.type = type;
        E[] enums = type.getEnumConstants();
        if (Objects.nonNull(enums)) {
            //这里将枚举值和枚举类型存入 enumMap
            for (E e : enums) {
                this.enumMap.put(e.toCode(), e);
            }
        }
    }

    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int index, E e, JdbcType jdbcType) throws SQLException {
        //这里将枚举的 code 转为数据库该字段的 int 类型
        preparedStatement.setInt(index, e.toCode());
    }

    @Override
    public E getNullableResult(ResultSet resultSet, String columnName) throws SQLException {
        //这里根据字段名来将数据库的 int 转为 Java 的 Integer
        Integer code = resultSet.getInt(columnName);
        if (resultSet.wasNull()){
            return null;
        }else {
            //取出对应的枚举值
            return enumMap.get(code);
        }
    }

    @Override
    public E getNullableResult(ResultSet resultSet, int columnIndex) throws SQLException {
        //这里根据字段位置来将数据库的 int 转为 Java 的 Integer
        Integer code = resultSet.getInt(columnIndex);
        if (resultSet.wasNull()){
            return null;
        }else {
            //取出对应的枚举值
            return enumMap.get(code);
        }
    }

    @Override
    public E getNullableResult(CallableStatement callableStatement, int columnIndex) throws SQLException {
        //这里根据SQL存储过程里的字段位置将字段的 int 转为 Java 的 Integer
        Integer code = callableStatement.getInt(columnIndex);
        if (callableStatement.wasNull()){
            return null;
        }else {
            //取出对应的枚举值
            return enumMap.get(code);
        }
    }

}


```


```
/**
 * 作用：该接口包含了两个枚举操作的抽象方法


 */
public interface MyEnum {

    /**
     * 根据 code 获取枚举实例
     * @param code
     */
    MyEnum fromCode(int code);

    /**
     * 获取枚举中的 code
     */
    int toCode();

}

```


```
@Getter
@RequiredArgsConstructor
public enum StudyStatusEnum implements MyEnum{
    ONE(1, "枚举1"),
    TWO(2, 枚举2"),
    THREE(3, "枚举3"),
    FOUR(4, "枚举4"),
    FIVE(5, "枚举5");

    private final Integer code;

    private final String desc;

    /**
     * 根据 code 获取枚举实例
     */
    @Override
    public MyEnum fromCode(int code) {
        return Arrays.stream(StudyStatusEnum.values())
                .filter(val -> val.getCode().equals(code))
                .findFirst().orElse(null);
    }

    /**
     * 获取枚举中的 code
     */
    @Override
    public int toCode() {
        return this.getCode();
    }

}

```



---


### 四、文章小结


通过内置和自定义的 TypeHandler，我们可以轻松处理各种数据类型转换需求，提高开发效率和代码可维护性。


在 Spring Boot 环境中使用自定义 TypeHandler 更是简化了配置和注册过程，使得我们能够更专注于业务逻辑的实现。


最后，文章如有不足和错误，还请大家指正。或者你有其它想说的，也欢迎大家在评论区交流！


