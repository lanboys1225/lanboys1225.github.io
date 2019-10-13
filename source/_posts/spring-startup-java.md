---
title: 如何手动启动Spring容器
date: 2019-10-12 23:12:02
category: spring
tags: [spring]
---

工作中，我想大家最熟悉的Spring容器启动方法，就是在web环境下，通过在web.xml中配置如下代码进行启动。

```
<context-param>
   <param-name>contextConfigLocation</param-name>
   <param-value>classpath*:/applicationContext.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

那么，离开了web环境，想单独的启动一个Spring容器该怎么做呢，其实也很简单，有两种方式，直接看代码：

###  1. 手动启动

目录结构：

![](https://pic.superbed.cn/item/5da1f344451253d178e6a6c6.png)

pom.xml

```
<properties>
    <spring.version>3.0.0.RELEASE</spring.version>
</properties>
<dependencies>
	<dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <!-- 日志依赖 -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.0.13</version>
    </dependency>
    <!-- 单元测试依赖 -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
        <scope>test</scope>
    </dependency>
    <!-- spring单元测试依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>${spring.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```
applicationContext.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
						http://www.springframework.org/schema/beans/spring-beans.xsd
						http://www.springframework.org/schema/context
						http://www.springframework.org/schema/context/spring-context.xsd">
						
    <bean id="helloWorld-id" name="helloWorld" class="com.bing.lan.spring.HelloWorld"/>
    
</beans>
```

日志配置： logback.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration debug="true">
    <contextName>spring</contextName>
    <property name="NORMAL_PATTERN"
              value=" %d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level - %logger{100} - %msg%n"/>
              
    <Appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <Layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>${NORMAL_PATTERN}</Pattern>
        </Layout>
    </Appender>

    <ROOT level="DEBUG">
        <Appender-ref ref="STDOUT"/>
    </ROOT>

</Configuration>
```

HelloWorld.java

```
public class HelloWorld {

    private String name = "OOPcoder";

    @Override
    public String toString() {
        return "HelloWorld{" +
                "name='" + name + '\'' +
                '}';
    }
}

```

SpringStartup.java

```
package com.bing.lan.spring;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class SpringStartup {

    public static void main(String[] args) {
        // 手动启动spring容器
        ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        
        HelloWorld helloWorld = (HelloWorld) context.getBean("helloWorld");
        System.out.println("main(): " + helloWorld);
    }
}
```

启动main函数，容器就启动了。

### 2. 通过 junit 来启动

在上面这些类的基础上再添加一个测试类

SpringTest.java

```
package com.bing.lan.spring;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:applicationContext.xml"})
public class SpringTest {

    @Autowired
    BeanFactory beanFactory;

    @Autowired
    ApplicationContext applicationContext;

    @Test
    public void test() {
        HelloWorld helloWorld = (HelloWorld) applicationContext.getBean("helloWorld");
        System.out.println("main(): " + helloWorld);

        helloWorld = (HelloWorld) beanFactory.getBean("helloWorld");
        System.out.println("main(): " + helloWorld);
    }
}
```

运行test()，容器启动成功。

学会了怎么启动，有啥好处呢，好处很多，比如

> 1. 可以脱离web环境测试我们的 service / mapper 层，极大的提高开发效率；
>
> 2. 还可以debug进Spring源码里学习各种原理，这对我们小白来说，是非常友好的，因为这只是一个单纯的Spring, 没有其他框架的干扰。