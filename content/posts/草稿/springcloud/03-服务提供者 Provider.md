---
title: "SpringCloud入门系列(三)---服务提供者 Provider"
description: "一个简单的服务提供者"
date: 2019-03-17 22:00:00
draft: true
tags: ["SpringCloud"]
categories: ["SpringCloud"]
---

本文主要介绍了`SpringCloud-Netflix`系列微服务解决方案之服务提供者。

<!--more-->

## 1. 概述

当 Eureka Client 向 Eureka Server 注册时，它会提供一些元数据，例如主机和端口，URL，主页等。Eureka Server 从每个 Client 实例接收心跳消息。 如果心跳超时，则通常将该实例从注册 Server 中删除。这与ZooKeeper注册中心很像。

## 2. 创建服务提供者项目

创建一个项目`hello-spring-cloud-service-admin`作为服务提供者。

### 2.1 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.lixueduan</groupId>
        <artifactId>hello-spring-cloud-dependencies</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../hello-spring-cloud-dependencies/pom.xml</relativePath>
    </parent>

    <artifactId>hello-spring-cloud-service-admin</artifactId>
    <packaging>jar</packaging>

    <name>hello-spring-cloud-service-admin</name>
    <url>http://www.lixueduan.com</url>
    <inceptionYear>2019-Now</inceptionYear>

    <dependencies>
        <!-- Spring Boot Begin -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Spring Boot End -->

        <!-- Spring Cloud Begin -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
        <!-- Spring Cloud End -->
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!--指向启动类 用于jar包方式运行-->  <mainClass>com.illusory.hello.spring.cloud.service.admin.ServiceAdminApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 2.2 Application

通过注解 `@EnableEurekaClient` 表明自己是一个 Eureka Client.

```java
package com.illusory.hello.spring.cloud.service.admin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient
public class ServiceAdminApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceAdminApplication.class, args);
    }
}
```

### 2.3 application.yml

```yml
spring:
  application:
    name: hello-spring-cloud-service-admin

server:
  port: 8762

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

**注意**： 需要指明 `spring.application.name`，这个很重要，这在以后的服务与服务之间相互调用一般都是根据这个` name`。

### 2.4 Controller

对外提供服务

```java
package com.illusory.hello.spring.cloud.service.admin.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AdminController {

    @Value("${server.port}")
    private String port;

    @RequestMapping(value = "hi", method = RequestMethod.GET)
    public String sayHi(@RequestParam(value = "message") String message) {
        return String.format("Hi，your message is : %s i am from port : %s", message, port);
    }
}
```

## 3. 小结

到这里一个简单的服务提供者就搭建成功了，接下来搭建一个服务消费者。