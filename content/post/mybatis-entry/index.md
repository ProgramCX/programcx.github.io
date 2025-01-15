---
title: MyBatis 入门
description: MyBatis 的基本使用
date: 2025-01-15
image: MyBatis.png
categories:
    - MyBatis
---

# 一、环境搭建

## 依赖引入

在pom.xml文件中添加以下代码：
```xml
<dependencies>  
    <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->  
    <dependency>  
        <groupId>org.mybatis</groupId>  
        <artifactId>mybatis</artifactId>  
        <version>3.5.19</version>  
    </dependency>    
    <!-- https://mvnrepository.com/artifact/com.mysql/mysql-connector-j -->  
    <dependency>  
        <groupId>com.mysql</groupId>  
        <artifactId>mysql-connector-j</artifactId>  
        <version>9.1.0</version>  
    </dependency>   
     <!-- https://mvnrepository.com/artifact/junit/junit -->  
    <dependency>  
        <groupId>junit</groupId>  
        <artifactId>junit</artifactId>  
        <version>4.13.2</version>  
        <scope>test</scope>  
    </dependency></dependencies>
```

## 创建数据库连接信息配置文件

在src/main/resources 目录下创建数据库连接的配置文件，命名为`db.properties`，用来配置数据库连接时的参数。
```properties
mysql.driver = com.mysql.cj.jdbc.Driver  
mysql.url = jdbc:mysql://localhost:3306/mall?serverTimezone=UTC&\characterEncoding=utf8&useUnicode=true&useSSL=false  
mysql.username = root  
mysql.password = mysql
```

## 创建 Mybatis 的核心配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE configuration  
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"  
        "http://mybatis.org/dtd/mybatis-3-config.dtd">  
<!-- 配置文件的根元素 -->  
<configuration>  
    <!-- 属性：定义配置外在化 -->  
    <properties resource="db.properties"/>  
    <!-- 环境：配置mybatis的环境 -->  
    <environments default="development">  
        <!-- 环境变量：可以配置多个环境变量，比如使用多数据源时，就需要配置多个环境变量 -->  
        <environment id="development">  
            <!-- 事务管理器 -->  
            <transactionManager type="JDBC"/>  
            <!-- 数据源 -->  
            <dataSource type="POOLED">  
                <property name="driver" value="${mysql.driver}"/>  
                <property name="url" value="${mysql.url}"/>  
                <property name="username" value="${mysql.username}"/>  
                <property name="password" value="${mysql.password}"/>  
            </dataSource>        
        </environment>    
    </environments>
</configuration>
```

# 二、MyBatis 入门程序

## 第一步：创建 POJO 实体

```java
package org.example.pojo;  
  
public class User {  
    private int userID;  
    private String userName;  
    private String sex;  
  
    public int getUserID() {  
        return userID;  
    }  
  
    public void setUserID(int userID) {  
        this.userID = userID;  
    }  
  
    public String getUserName() {  
        return userName;  
    }  
  
    public void setUserName(String userName) {  
        this.userName = userName;  
    }  
  
    public String getSex() {  
        return sex;  
    }  
  
    public void setSex(String sex) {  
        this.sex = sex;  
    }  
}
```


POJO 实体用来从数据库映射的值。

## 第二步：创建映射文件 UserMapper.xml

### 文件命名

文件命名通常使用**POJO 实体类名+Mapper 命名**。例如User 实体类映射文件名为    **UserMapper.xml** 。

```xml
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="org.example.pojo.User">  
    <select id="findUserById" parameterType="int" resultType="org.example.pojo.User">  
        select * from user where userID = # {id}  
    </select>  
</mapper>
```

## 第三步：在 mybatis-config.xml 配置文件里面添加映射

```xml
<mappers>  
    <mapper resource="mapper/UserMapper.xml"/>  
</mappers>
```
 此代码在`<configuration><configuration/>` 里面直接添加。

## 第四步：编写测试类

在 test/java/Test 下创建 UserTest.java 文件。
```java
package Test;  
  
import org.example.pojo.User;  
import org.apache.ibatis.io.Resources;  
import org.apache.ibatis.session.SqlSession;  
import org.apache.ibatis.session.SqlSessionFactory;  
import org.apache.ibatis.session.SqlSessionFactoryBuilder;  
import org.junit.Test;  
import java.io.IOException;  
import java.io.Reader;  
public class UserTest {  
    @Test  
    public void test() throws IOException {  
  
       //第一步：将 mybatis-config.xml 文件里面的内容加载到 reader 对象中  
        Reader reader = Resources.getResourceAsReader("mybatis-config.xml");  
  
        //第二步：初始化MyBatis数据库，创建 SqlSessionFactory 类的实例  
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);  
  
        //第三步：创建SqlSession会话实例  
        SqlSession sqlSession = sqlSessionFactory.openSession();  
  
        //获取查询到的结果映射到的对象  
        User user = sqlSession.selectOne("findUserById", 1);  
        System.out.println(user.getUserName());  
        sqlSession.close();  
    }  
}
```

在上述代码中，先通过`Resource`的`getResourceReader()`方法来将 MyBatis 配置文件**加载到 reader 对象中**，接着将read传入 SqlSessionFactoryBuilder().build() 工厂方法来**构建SqlSessionFactory 类的对象**。之后这个对象调用openSession方法来**获取一个SqlSession会话**。然后调用selectOne()来**获取查询到的结果映射到的对象**。最后就可以**用Get方法来获取**想得到的数据了。

运行测试，得到结果：
```console
ChengXu
```
