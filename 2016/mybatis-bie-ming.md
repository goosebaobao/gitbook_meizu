# Mybatis 别名

## 用途

别名就是完整的类名（即包含完整包名的类名）的一个简写，可以在 mybatis 的 sql 映射文件里用别名来代替完整的类名，简化书写

## 定义别名

在 mybatis 配置文件里使用 typeAlias 节点来定义别名，示例如下

```markup
<typeAliases>
    <typeAlias alias="Menu" type="com.meizu.gameissue.boss.domain.entity.Menu" />
</typeAliases>
```

另一种方法是定义整个包的所有类的别名

```markup
<typeAliases>
    <package name="com.meizu.gameissue.boss.domain.entity"/>
</typeAliases>
```

mybatis 会扫描包下的所有类（不包括内部类和接口），并为每个类设定一个别名

* 如果该类使用了`@Alias`注解，那么用注解值作为别名
* 没有使用注解，用这个类的类名（首字母小写）作为别名
* 如果用 typeAlias 定义了别名，`@Alias`注解定义了一个不同的别名？不要问我，我不知道。。。

## 使用别名

在 sql 映射文件里，用别名来代替完整的类名，示例如下

```markup
<update id="update" parameterType="Menu">
    update T_AUTH_MENU set FID=#{id}
    <if test="name!=null">
        , FNAME=#{name}
    </if>
    <if test="url!=null">
        , FURL=#{url}
    </if>
    <if test="lvl!=-1">
        , FLVL=#{lvl}
    </if>
    <if test="parentId!=null">
        , FPARENTID=#{parentId}
    </if>
    <if test="visible!=null">
        , FVISIBLE=#{visible}
    </if>
    <if test="order!=null">
        , FORDER=#{order}
    </if>
    where FID=#{id}
</update>
```

## 预定义的别名

mybatis 已经预先定义了常用类的别名，我们可以直接使用这些别名而不需要定义，同时也要注意，在定义自己的别名时也不要和这些别名冲突

mybatis 预定义了哪些别名呢？

```java
public TypeAliasRegistry() {
  registerAlias("string", String.class);

  registerAlias("byte", Byte.class);
  registerAlias("long", Long.class);
  registerAlias("short", Short.class);
  registerAlias("int", Integer.class);
  registerAlias("integer", Integer.class);
  registerAlias("double", Double.class);
  registerAlias("float", Float.class);
  registerAlias("boolean", Boolean.class);

  registerAlias("byte[]", Byte[].class);
  registerAlias("long[]", Long[].class);
  registerAlias("short[]", Short[].class);
  registerAlias("int[]", Integer[].class);
  registerAlias("integer[]", Integer[].class);
  registerAlias("double[]", Double[].class);
  registerAlias("float[]", Float[].class);
  registerAlias("boolean[]", Boolean[].class);

  registerAlias("_byte", byte.class);
  registerAlias("_long", long.class);
  registerAlias("_short", short.class);
  registerAlias("_int", int.class);
  registerAlias("_integer", int.class);
  registerAlias("_double", double.class);
  registerAlias("_float", float.class);
  registerAlias("_boolean", boolean.class);

  registerAlias("_byte[]", byte[].class);
  registerAlias("_long[]", long[].class);
  registerAlias("_short[]", short[].class);
  registerAlias("_int[]", int[].class);
  registerAlias("_integer[]", int[].class);
  registerAlias("_double[]", double[].class);
  registerAlias("_float[]", float[].class);
  registerAlias("_boolean[]", boolean[].class);

  registerAlias("date", Date.class);
  registerAlias("decimal", BigDecimal.class);
  registerAlias("bigdecimal", BigDecimal.class);
  registerAlias("biginteger", BigInteger.class);
  registerAlias("object", Object.class);

  registerAlias("date[]", Date[].class);
  registerAlias("decimal[]", BigDecimal[].class);
  registerAlias("bigdecimal[]", BigDecimal[].class);
  registerAlias("biginteger[]", BigInteger[].class);
  registerAlias("object[]", Object[].class);

  registerAlias("map", Map.class);
  registerAlias("hashmap", HashMap.class);
  registerAlias("list", List.class);
  registerAlias("arraylist", ArrayList.class);
  registerAlias("collection", Collection.class);
  registerAlias("iterator", Iterator.class);

  registerAlias("ResultSet", ResultSet.class);
}
```

注意

* 预定义别名，大小写不敏感
* 对象类型和原始类型是有区别的，别名不一样

## 参考文档

* [mybatis 官方中文文档](http://www.mybatis.org/mybatis-3/zh/)



