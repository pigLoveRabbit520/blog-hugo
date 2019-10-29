title: Spring Boot集成MyBatis操作MySQL
author: Salamander
tags:
  - Spring
  - Spring Boot
  - MyBatis
categories:
  - Java
date: 2019-10-27 19:05:00
---
最近学习了一下Spring Boot，它确实做到了简单快速创建Java Web应用。这是一篇简单的笔记，记录了Spring Boot集成MyBatis，实现基本的CURD。

# MyBatis集成方式
* 注解版集成
* XML版本集成

注解方式集成比较简洁，本文就不作介绍。  
XML集成是比较老的集成方式，以前配SSM（Spring+SpringMVC+MyBatis）用的就是这种方式。



<!-- more -->

XML版本为老式的配置集成方式，重度集成XML文件，SQL语句也是全部写在XML中的；注解版版本，相对来说比较简约，不需要XML配置，只需要使用注解和代码来操作数据。
## 准备
启动MySQL服务  
创建数据库`spring_db`
```
CREATE DATABASE spring_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

创建`user`表
```
create table user
(
	uid int(11) unsigned auto_increment comment '主键Id'
		primary key,
	name varchar(255) null comment '名称',
	age int null comment '年龄',
	address varchar(255) null comment '地址',
	created_time datetime null comment '创建时间',
	updated_time datetime null comment '更新时间'
)
comment '用户表' collate=utf8_general_ci;
```


## 添加依赖
```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.41</version>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.25</version>
</dependency>
```

## 配置数据库连接
设置application.properties文件，添加如下配置
```
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/spring_db?useUnicode=true&characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=2LCqvSOJ6m0Ut6ui
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```
* spring.datasource.url 数据库连接字符串
* spring.datasource.username 数据库用户名
* spring.datasource.password 数据库密码
* spring.datasource.driver-class-name 驱动类型（注意MySQL 8.0的值是com.mysql.cj.jdbc.Driver和之前不同）

## 设置 MapperScan 包路径
直接在启动文件SpringbootApplication.java的类上配置@MapperScan，这样就可以省去，单独给每个Mapper（就是我们这里的dao层）上标识@Mapper的麻烦。
```
@SpringBootApplication
@MapperScan("com.salamander.springbootdemo.dao")
public class SpringbootdemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootdemoApplication.class, args);
    }
}
```

## 添加Entity和Dao层类
`com.salamander.springbootdemo.entity`下`User`类（使用了lombok的@Data注解）：
```
@Data
public class User implements Serializable {
    private Long id;

    private String name;

    private String address;

    private int age;

    private Date createdDatetime;
}
```
`com.salamander.springbootdemo.dao`下`UserDao`接口：
```
public interface UserDao {
    User findByName(String name);
}
```

### XML方式MyBatis 集成
修改application.properties，添加配置
```
mybatis.config-locations=classpath:mybatis/mybatis-config.xml
mybatis.mapper-locations=classpath:mybatis/mapper/*.xml
```
* mybatis.config-locations 配置MyBatis基础属性
* mybatis.mapper-locations 配置Mapper XML文件

## 配置XML文件
本例创建两个xml文件，在resource/mybatis下的mybatis-config.xml（配置MyBatis基础属性）和在resource/mybatis/mapper下的UserMapper.xml（用户和数据交互的SQL语句）。
`mybatis-config.xml`
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <typeAliases>
        <typeAlias alias="Integer" type="java.lang.Integer"/>
        <typeAlias alias="Long" type="java.lang.Long"/>
        <typeAlias alias="HashMap" type="java.util.HashMap"/>
        <typeAlias alias="LinkedHashMap" type="java.util.LinkedHashMap"/>
        <typeAlias alias="ArrayList" type="java.util.ArrayList"/>
        <typeAlias alias="LinkedList" type="java.util.LinkedList"/>
    </typeAliases>
</configuration>
```

`UserMapper.xml`
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--namespace是命名空间，是dao接口的全路径-->
<mapper namespace="com.salamander.springbootdemo.dao.UserDao">
    <resultMap id="userResultMap" type="com.salamander.springbootdemo.entity.User">
        <id property="id" column="uid"></id>
        <result column="name" property="name" />
        <result column="age" property="age" />
        <result column="address" property="address" />
        <result column="created_time" property="createdDatetime" />
    </resultMap>


    <select id="findByName" parameterType="java.lang.String"  resultMap="userResultMap">
        select  uid, name, age, address, created_time
        from  user
        where name = #{name}
    </select>
</mapper>
```

## 调用Dao类
`HomeController.java`类
```
@RestController
public class HomeController {
    @Resource
    private UserDao userDao;


    @RequestMapping("/")
    public String index() {
        return "Hello World!";
    }


    @RequestMapping("/user/{username}")
    @ResponseBody
    public User getUser(@PathVariable(name = "username") String name) {
        return userDao.findByName(name);
    }
}
```
好了，访问链接`http://localhost:8080/user/wang`，就会输出`wang`这个用户的数据



项目代码[下载](http://file.51lucy.com/SpringBootDemo.zip)