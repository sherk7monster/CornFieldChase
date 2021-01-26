---
title:  "SpringBoot-创建自定义starter"
category: "SpringBoot"
---

2021-01-26 记

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
<span id="busuanzi_container_page_pv">
  阅读量&nbsp;<span id="busuanzi_value_page_pv"></span>&nbsp;次，
</span>本文约 {{ content | strip_html | strip_newlines | split: "" | size }} 字

目录
* 目录
{:toc}

## Starter 定义

**Spring 官网定义：** Starters are a set of convenient dependency descriptors that you can include in your application. You get a one-stop shop for all the Spring and related technologies that you need without having to hunt through sample code and copy-paste loads of dependency descriptors. 
{: .notice--info}

大意是说 `Starters` 提供了一些应用依赖的打包合集，从而开发人员在开发时不需要一个个的去手动添加依赖，做到开箱即用。 `Spring` 默认提供了一些常见的像 `SpringMVC` 的组件 starter 如 `spring-boot-starter-web`,还有日志 `spring-boot-starter-logging` 等。

## 创建自己的 Starter

一个自定义的 `starter` 本身也是一个依赖，但是这个 `starter` 不能以 `spring-boot` 开头命名，无论 Maven 的 `groupId` 是否和 Spring 其他项目是否不一样。如果你的 `starter` 还提供了配置 `key` ，那你的 key 前缀定义也必须是要唯一的，并且不能和 Spring 内置的配置 key 冲突，比如 `server`、`spring` 等。对于每一个自定义的配置 key 对应的字段，也最好有 Java 注释。
{: .notice--info}

这里我们以实现一个自动注入的 `Student` 的实例为例，用户通过在 Spring 的 `application.properties` 中配置 Student 的属性值，实现自动注入。

### 第一步，引入必要的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

该依赖使我们可以配置 starter 的 `SPI` ，并且编译程序后能生成 `spring-configuration-metadata.json` 文件，该文件是给 IDE 提供代码补全功能用的。如下图所示：

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/codetab.jpg){: .align-center}

### 第二步，编写属性配置类和待注入的 Student 类

##### 属性配置类
```java
package test.starter.prop;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import java.util.Properties;

/**
 * Spring boot properties configuration.
 */
@ConfigurationProperties(prefix = "test.starter")
@Getter
@Setter
public final class DomainPropertiesConfiguration {

    /**
     * config field, named `props`
     */
    private Properties props = new Properties();

}
```

`prefix` 代表的就是你的配置 key 的前缀，`props` 本身就是配置 key 的一个字段名。

##### Student 类

```java
package test.starter.domain;

import lombok.Data;
import lombok.ToString;

@Data
@ToString
public class Student {
    private int id;
    private String name;
}
```

### 第三步，编写 SpringBoot 配置类

```java
package test.starter;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import test.starter.domain.Student;
import test.starter.prop.DomainPropertiesConfiguration;
import javax.annotation.Resource;

@Configuration
@ComponentScan("test.starter.domain")
@EnableConfigurationProperties(DomainPropertiesConfiguration.class)
@RequiredArgsConstructor
public class TestSpringBootConfiguration {
    private final DomainPropertiesConfiguration props;

    @Bean
    public Student student(){
        Student student=new Student();
        student.setId(Integer.parseInt(props.getProps().getProperty("student.id")));
        student.setName(props.getProps().getProperty("student.name"));
        return student;
    }
}
```

### 第四步，提供 SPI 相关文件

最后还需提供2个文件：`/META-INF/spring.factories` 和 `/META-INF/spring.provides`

##### /META-INF/spring.factories
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
test.starter.TestSpringBootConfiguration
```

##### /META-INF/spring.provides
```properties
# provides后面就填你自定义的 starter 的项目名就行
provides: test-spring-boot-starter
```

## 测试自定义 Starter

### 创建测试的 demo 项目

##### 引入自定义的 Starter 依赖

```xml
<dependency>
    <groupId>test.custom.starter</groupId>
    <artifactId>test-spring-starter</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

##### 在 application.properties 中注入 Student 的属性值

```properties
test.starter.props.student.id=1
test.starter.props.student.name=test
```

##### 编译项目，输出 Student 的值

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;
import test.starter.domain.Student;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context=SpringApplication.run(DemoApplication.class, args);
        Student student= (Student) context.getBean("student");
        System.out.println(student);
    }

}
```

结果如下：
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/testconsole20210126.jpg){: .align-center}

## 最后

看一下编译后的 classes 文件，`META-INF` 目录下多了一个 `spring-configuration-metadata.json` 文件
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/testconsole2021012602.jpg){: .align-center}

这个文件就是前面说的，给 IDE 实现自动提示用的。我们看下 `spring-configuration-metadata.json` 的文件内容：

```json
{
  "hints": [],
  "groups": [
    {
      "sourceType": "test.starter.prop.DomainPropertiesConfiguration",
      "name": "test.starter",
      "type": "test.starter.prop.DomainPropertiesConfiguration"
    }
  ],
  "properties": [
    {
      "sourceType": "test.starter.prop.DomainPropertiesConfiguration",
      "name": "test.starter.props",
      "description": "config field, named `props`",
      "type": "java.util.Properties"
    }
  ]
}
```

