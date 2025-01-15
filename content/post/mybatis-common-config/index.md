---
title: MyBatis 常用配置
description: MyBatis 的常用配置
date: 2025-01-16
image: MyBatis.jpg
slug: MyBatis
categories:
    - MyBatis
---

# MyBatis 中核心对象

## 1. SqlSession

SqlSession 中提供了不同的查询方法。其中可以查询单个，也可以查询符合查询条件的集合。不但可以进行查询操作，也可以进行增删改的功能。下面介绍几个常用的方法：

1.  `<T> T selectOne(String statement, Object parameter)` :  parameter 是语句所需要的参数，用来将**参数传递给 SQL 语句**。该方法会返回一个xml映射配置中指定的类型。
2. `<E> List<E> selectList(String statement, Object parameter)` :**查询泛型对象的集合**。
3.  `int insert(String statement, Object parameter)` ：插入方法，参数`statement` 是`<insert>` 的`id`，`parameter` 是插入语句所需要的参数，也就是后面要学到的`parameterType`
4. `int update(String statement, Object parameter)`,`int delete(String statement, Object parameter)` 用法类似。
5. `void commit()`：提交事务。
6. `void rollback()`：回滚事务。
7. `void close()`：关闭`SqlSession`对象。

## 2. MyBatis 核心配置文件元素

### 2.2 `<properties>`元素
用来存放数据库连接驱动类型、sql地址、用户名和密码。然后在`mybatis-cnfig.xml`里面引入、使用。

### 2.3 `<typeAlias>`元素

给全限定名起一个别名(Alias)。

在`mybatis-config.xml`文件里面加上：
```xml
<typeAliases>
	<typeAlias alias="User" type="org.example.pojo.User" />
</typeAliases>
```

### 2.4 `<environments>`元素

**可以给开发环境、生产环境进行不同的配置。**

- UNPOOLED：无连接池类型。每次请求会打开和关闭数据库。适用于对性能要求不高的简单应用程序。
- POOLED：连接池类型。
### 2.6 `<mappers>`元素

用来引入映射文件。基本结构：
```xml
<mappers>  
    <mapper resource="mapper/UserMapper.xml"/>  
</mappers>
```
  

### 2.7 ``<select>``元素

  

该标签可以实现查询操作，可以将查询结果映射为POJO对象，并且返回。

简单的查询例子：

  

```xml

    <select id="findUserById" parameterType="int" resultType="cn.programcx.POJO.User" >

         SELECT * FROM user WHERE id = #{id}

    <select>

```

  

**这个语句接受一个int类型的参数，返回一个``cn.programcx.POJO``包中的对象。**

  

参数：

| 属性            | 描述                                                                                                              |
| :------------ | :-------------------------------------------------------------------------------------------------------------- |
| id            | 在命名空间中唯一的标识符，可以被用来引用这条语句。                                                                                       |
| parameterType | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。                     |
| resultType    | 期望从这条语句中返回结果的类全限定名或别名。注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 resultType 和 resultMap 之间只能同时使用一个。                  |
| resultMap     | 期望从这条语句中返回结果的类全限定名或别名。注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 resultType 和 resultMap 之间只能同时使用一个。                  |
| flushCache    | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false。                                                                |
| useCache      | 将其设置为 true 后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 select 元素为 true。                                                        |
| timeout       | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。                                                        |
| fetchSize     | 这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个设置值。默认值为未设置（unset）（依赖驱动）。                                                     |
| statementType | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 |
| resultSetType | FORWARD_ONLY，SCROLL_SENSITIVE, SCROLL_INSENSITIVE 或 DEFAULT（等价于 unset）中的一个，默认值为 unset（依赖数据库驱动）。                 |
| databaseId    | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。          |
| resultOrdered | 这个设置仅针对嵌套结果 select 语句：如果为 true，将会假设包含了嵌套结果集或是分组，当返回一个主结果行时，就不会产生对前面结果集的引用。默认值：false。                            |
| resultSets    | 这个设置仅适用于多结果集的情况。它将列出语句执行后返回的结果集并赋予每个结果集一个名称，多个名称之间以逗号分隔。                                                        |


  

**注意：如果映射的POJO对象和指定查询的数据表中名称不一致，不能使用``resultType``，否则无法进行映射，会抛出异常；如果名称不一致，应使用[resultMap](#section1)（后面介绍）**。

### 2.8 ``<insert>``元素

insert标签用来指定插入记录的操作和属性。

```xml

    <insert id="insertUser" parameterType="cn.programcx.POJO" useGeneratedKeys="true" keyProperty="id">

        INSERT INTO user(name,sex,email,phone,address,password) values (#{name},#{sex},#{email},#{phone},#{address},#{password})

    </insert>

```

其中，``#{name}``表示映射的POJO对象的成员变量·``name``。


| 属性               | 描述                                                                                                                                           |
| :--------------- | :------------------------------------------------------------------------------------------------------------------------------------------- |
| id               | 在命名空间中唯一的标识符，可以被用来引用这条语句。                                                                                                                    |
| parameterType    | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。                                                  |
| parameterMap     | 用于引用外部 parameterMap 的属性，目前已被废弃。请使用行内参数映射和 parameterType 属性。                                                                                  |
| flushCache       | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：（对 insert、update 和 delete 语句）true。                                                                 |
| timeout          | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。                                                                                     |
| statementType    | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。                              |
| useGeneratedKeys | （仅适用于 insert 和 update）这会令 MyBatis 使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键（比如：像 MySQL 和 SQL Server 这样的关系型数据库管理系统的自动递增字段），默认值：false。       |
| keyProperty      | （仅适用于 insert 和 update）指定能够唯一识别对象的属性，MyBatis 会使用 getGeneratedKeys 的返回值或 insert 语句的 selectKey 子元素设置它的值，默认值：未设置（unset）。如果生成列不止一个，可以用逗号分隔多个属性名称。 |
| keyColumn        | （仅适用于 insert 和 update）设置生成键值在表中的列名，在某些数据库（像 PostgreSQL）中，当主键列不是表中的第一列的时候，是必须设置的。如果生成列不止一个，可以用逗号分隔多个属性名称。                                     |
| databaseId       | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。                                       |
|                  |                                                                                                                                              |

  

### 2.9 ``<update>``元素

用来更新记录。

```xml

    <update id="updateUser" parameterType="cn.programcx.POJO.user" >

        UPDATE user SET name = #{name},email = #{email},password = #{password} WHERE userId = #{userId}

    </update>

```

  

属性和上面的``insert``一样。

  

### 2.10 ``<delete>``元素

用来删除记录

```xml

    <delete id="deleteUser" ></delete>

        DELETE FROM user WHERE id=#{id}

    </delete>

```

### 2.11 `<selectKey>`元素

用来自定义生成一个属性的值。

  

例如，插入用户为用户随机生成id并插入数据表：

``` xml

<insert id="insertUser" parameterType="cn.programcx.POJO.user">

    <selectKey keyProperty="userId" resultType="int" order="BEFORE">

     select CAST(RANDOM()*1000000 as INTEGER) a from SYSIBM.SYSDUMMY1

     </selectKey>

        INSERT INTO user(userId,name,sex,email,phone,address,password)

        values (#{userId},#{name},#{sex},#{email},#{phone},#{address},#{password})

</insert>

```

### 2.11 `<resultMap>`元素

`<resultMap>`的作用是自定义映射规则。如果数据表字段名称和POJO类中所需要映射到属性的名称不一致，就不能仅仅指定`parameterType`或``resultType``来实现映射，必须要通过自定义映射来实现。
其基本格式是：
```xml
<resultMap type="[完整类名]" id="[标识这个自定义映射的字符串]">
	<id property="[POJO类中的属性名称]" column="[字段名称]"/>
	<result property="[POJO类中的属性名称]" column="[字段名称]"/>
</resultMap>
```