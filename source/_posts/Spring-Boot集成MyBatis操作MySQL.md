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

## MyBatis集成方式
* 注解版集成
* XML版本集成

XML版本为老式的配置集成方式，重度集成XML文件，SQL语句也是全部写在XML中的，我以前配SSM（Spring+SpringMVC+MyBatis）用的就是这种方式；注解版版本，相对来说比较简约，不需要XML配置，只需要使用注解和代码来操作数据，本文这里不作介绍（其实挺好学的，^_^）。 



<!-- more -->

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
    
    int insertUser(User user);
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
    
    <insert id="insertUser" parameterType="com.salamander.springbootdemo.entity.User">
        insert into user(name, age, address, created_time) VALUES (
        #{name}, #{age}, #{address}, #{createdDatetime}
        )
    </insert>
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
    
    @RequestMapping("/user/add/{username}")
    @ResponseBody
    public String addUser(@PathVariable(name = "username") String name) {
        User user = new User();
        user.setName(name);
        user.setAge(20);
        user.setCreatedDatetime(new Date());
        userDao.insertUser(user);
        return "insert succesfully";
    }
}
```
好了，访问链接`http://localhost:8080/user/wang`，就会输出`wang`这个用户的数据，而访问`http://localhost:8080/user/add/zhao`,会添加一条name为`zhao`的数据到数据库。


## 事务支持

在SpringBoot中开启事务非常简单，只需在业务层添加事务注解(`@Transactional`)即可快速开启事务。好的，让我们来尝试一下。   
在上面的使用中，我们是直接把`Dao`类在控制层中使用的，但一般情况下，我们是在业务层中使用`Dao`类的。  
在`com.salamander.springbootdemo`下新建`Service`的package，之后创建`接口`UserService：
```
package com.salamander.springbootdemo.service;

public interface UserService {
    void addUsers(String name) throws Exception;
}
```
之后在`impl`的子package中添加实现类`UserServiceImpl`：
```

package com.salamander.springbootdemo.service.impl;

import com.salamander.springbootdemo.dao.UserDao;
import com.salamander.springbootdemo.entity.User;
import com.salamander.springbootdemo.service.UserService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.annotation.Resource;
import java.util.Date;

@Service
public class UserServiceImpl implements UserService {
    @Resource
    private UserDao userDao;


    @Transactional
    @Override
    public void addUsers(String name) throws Exception {
        int num = 5;
        for (int i = 0; i < num; i++) {
            User user = getNewUser(name + (i + 1));
            userDao.insertUser(user);
            if (i == 3) {
                throw new Exception("发生内部错误了");
            }
        }
    }

    private User getNewUser(String name) {
        User user = new User();
        user.setName(name);
        user.setAge(20);
        user.setCreatedDatetime(new Date());
        return user;
    }
}
```
然后我们在`HomeController`中注入`UserService`，并添加路由
```
@Resource
private UserService userService;

@RequestMapping("/users/add/{username}")
@ResponseBody
public String addUsers(@PathVariable(name = "username") String name) {
    try {
        userService.addUsers(name);
        return "batch insert succesfully";
    } catch (Exception e) {
        return e.getMessage();
    }
}

```

可以看到，我们在`addUsers`方法上添加了`@Transactional`注解开启了事务，并在插入第4条数据后抛出了异常。好了，让我们访问链接`http://localhost:8080/users/add/sun`，我们发现数据库多出了四条`name`为`sun`的数据，**回滚并没有起效果**  
![](https://s2.ax1x.com/2019/11/03/KXiGOe.png)

这是一个常见的坑点，因为`Spring`的默认的事务规则是遇到**运行异常**（`RuntimeException`及其子类）和程序错误（Error）才会进行事务回滚，而`Exception`是基类就不行了，让我们看下Java的异常类层次图  
![](https://s2.ax1x.com/2019/11/03/KXkgGq.jpg)
如果想针对检测异常进行事务回滚，可以在`@Transactional`注解里使用
`rollbackFor`属性明确指定异常（或者你可以自己定义一个继承`RuntimeException`的类，然后抛出这个类）。  
现在`addUsers`改成这样，就可以正常回滚了：
```
@Transactional(rollbackFor = Exception.class)
@Override
public void addUsers(String name) throws Exception {
    int num = 5;
    for (int i = 0; i < num; i++) {
        User user = getNewUser(name + (i + 1));
        userDao.insertUser(user);
        if (i == 3) {
            throw new Exception("发生内部错误了");
        }
    }
}
```


项目代码[下载](http://file.51lucy.com/SpringBootDemo.zip)