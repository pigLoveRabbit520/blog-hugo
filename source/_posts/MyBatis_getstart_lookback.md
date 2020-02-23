title: MyBatis经典入门回顾
author: Salamander
tags:
  - MyBatis
categories:
  - java
date: 2020-02-21 12:00:00
---
![](https://mybatis.org/images/mybatis-logo.png)

其实，无论是Mybatis、Hibernate都是ORM的一种实现框架，都是对**JDBC**的一种封装。 

之前我写过一篇[Spring Boot集成MyBatis操作MySQL](2019/10/27/Spring-Boot集成MyBatis操作MySQL/)，不过在这里让我们脱离Spring（不过很多代码是一样的,Dao类，Model类，数据库配置），就单独回顾下MyBatis的使用，来了解下MyBatis的使用流程。  

## Mybatis工作流程

* 通过Reader对象读取Mybatis配置文件
* 通过SqlSessionFactoryBuilder对象创建SqlSessionFactory对象
* 获取当前线程的SQLSession
* 事务默认开启
* 通过SQLSession读取映射文件中的操作编号，从而读取SQL语句
* 提交事务
* 关闭资源

<!-- more -->

**SqlSessionFactory**是一个很重要的类，每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为核心的。SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先定制的 Configuration 的实例构建出 SqlSessionFactory 的实例。

## pom.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.51lucy.test</groupId>
    <artifactId>mybatis-simple</artifactId>
    <version>1.0</version>


    <dependencies>
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.3</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.41</version>
        </dependency>
    </dependencies>

</project>
```

## 数据库配置jdbc.properties
在**src/main/resources**文件夹下创建`jdbc.properties`文件：
```
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/spring_db
jdbc.username=root
jdbc.password=***********
```


## mybatis配置文件
在**src/main/resources**文件夹下创建`mybatis-config.xml`文件：
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="jdbc.properties"/>

    <environments default="development">
        <!-- 设置一个默认的连接环境信息 -->
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driverClassName}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="mappers/User.xml"/>
    </mappers>
</configuration>
```


## 添加Dao接口
UserDao接口（其实就是Mapper接口）
```
package com.lucy.test.dao;

import com.lucy.test.model.User;

public interface UserDao {
    User findByName(String name);

    int insertUser(User user);
}

```

## xml映射文件
在**src/main/resources/mappers**文件夹下创建`User.xml`文件：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--namespace要写对-->
<mapper namespace="com.lucy.test.dao.UserDao">
    <select id="findByName" parameterType="java.lang.String"  resultType="com.lucy.test.model.User">
        select  uid, name, age, address, created_time
        from  user
        where name = #{name}
    </select>

    <insert id="insertUser" parameterType="com.lucy.test.model.User">
        insert into user(name, age, address, created_time) VALUES (
        #{name}, #{age}, #{address}, #{createdDatetime}
        )
    </insert>
</mapper>

```

## 测试类
让我们写个测试类看看效果：
```
package com.lucy.test;

import com.lucy.test.dao.UserDao;
import com.lucy.test.model.User;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;

public class Main {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            UserDao userMapper = sqlSession.getMapper(UserDao.class);
            User user = userMapper.findByName("meng");
            System.out.println(user.toString());
            sqlSession.commit();
        } catch (Exception ex) {
            ex.printStackTrace();
        } finally {
            sqlSession.close();
        }

    }
}
```

参考：
* [MyBatis Tutorial – CRUD Operations and Mapping Relationships – Part 1
](https://www.javacodegeeks.com/2012/11/mybatis-tutorial-crud-operations-and-mapping-relationships-part-1.html)