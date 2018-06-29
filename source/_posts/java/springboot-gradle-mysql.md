---
title: 'springboot+gradle下访问mysql用例'
date: 2018-06-29 21:05:46
tags: java
---

参考资料： [https://blog.csdn.net/chentao866/article/details/67636520](https://blog.csdn.net/chentao866/article/details/67636520)

## 配置依赖build.gradle

```
dependencies {
	compile('org.springframework.boot:spring-boot-starter-web')
	runtime('org.springframework.boot:spring-boot-devtools')
	compile 'org.springframework.boot:spring-boot-starter-data-jpa'
	runtime("mysql:mysql-connector-java")
	testCompile('org.springframework.boot:spring-boot-starter-test')
}
```
<!--more-->

## 配置mysql连接属性application.properties

假设已经创建好了mysql表test::dept，并且已插入部分测试数据，

```
server.port=8004

spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:13306/test?useUnicode=true&useSSL=false
spring.datasource.username=root
spring.datasource.password=root
 
spring.jpa.database=mysql
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
#spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.hibernate.naming-strategy=org.hibernate.cfg.ImprovedNamingStrategy
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5Dialect
```

## 部门表实体类Dept.java

```java
package com.example.demo;


import javax.persistence.*;

import org.hibernate.annotations.GenericGenerator;
 
 
/**
 * 部门表实体类
 */
@Entity
@Table(name = "dept")
public class Dept {
 
    //部门编号 主键
    @Id
    @Column(name = "id")
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    private String id;
 
    //部门编码
    @Column(name = "code")
    private String code;
 
    //部门名称
    @Column(name = "name")
    private String name;
 
    public String getId() {
        return id;
    }
 
    public void setId(String id) {
        this.id = id;
    }
 
    public String getCode() {
        return code;
    }
 
    public void setCode(String code) {
        this.code = code;
    }
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
}
```

## 仓库接口实现类DeptRepository.java

```java
package com.example.demo;

import org.springframework.data.jpa.repository.JpaRepository;
 
/**
 * 实现Jpa仓库接口类 
 */
public interface DeptRepository extends JpaRepository<Dept, String> {
}
```

## 控制类DeptController.java

```java
package com.example.demo;  

import java.util.*;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMethod;

import org.springframework.beans.factory.annotation.Autowired;
 
@RestController
public class DeptController {
    //部门Jpa仓库接口
    @Autowired
    private DeptRepository deptRepository;
 
    @RequestMapping(value="/deptfind", method = RequestMethod.GET)
    public List<Dept> deptfind() {
        List<Dept> deptList = deptRepository.findAll();
        for (int i = 0; i < deptList.size(); i++) {
            System.out.println(deptList.get(i));
        }
        return deptList;
    }
}
```

## 测试

具体代码结构如下图所示：
![](/img/deptstruct.png)

```shell
$./gradlew bootRun

> Task :bootRun
// ... 省略
```
服务起来后，通过浏览器访问：
![](/img/depttest.png)
