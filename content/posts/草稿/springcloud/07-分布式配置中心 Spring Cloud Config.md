---
title: "pringCloud入门系列(七)---分布式配置中心 Spring Cloud Config"
description: "分布式配置中心 Spring Cloud Config"
date: 2019-03-21 22:00:00
draft: true
tags: ["SpringCloud"]
categories: ["SpringCloud"]
---

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。

<!--more-->

##  1. 概述

在 Spring Cloud 中，有分布式配置中心组件 Spring Cloud Config ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程 Git 仓库中。在 Spring Cloud Config 组件中，分两个角色，一是 Config Server，二是 Config Client。

## 2. 搭建分布式配置中心服务端

创建一个工程名为 `hello-spring-cloud-config` 的项目

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

    <artifactId>hello-spring-cloud-config</artifactId>
    <packaging>jar</packaging>

    <name>hello-spring-cloud-config</name>
    <url>http://www.lixueduan.com</url>
    <inceptionYear>2019-Now</inceptionYear>

    <dependencies>
        <!-- Spring Boot Begin -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Spring Boot End -->

        <!-- Spring Cloud Begin -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
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
                    <mainClass>com.illusory.hello.spring.cloud.config.ConfigApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

主要增加了 `spring-cloud-config-server` 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

### 2.2 Application

通过 `@EnableConfigServer` 注解，开启配置服务器功能

```java
package com.illusory.hello.spring.cloud.config;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableConfigServer
@EnableEurekaClient
public class ConfigApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }
}
```

### 2.3 application.yml

增加 Config 相关配置，并设置端口号为：`8888`

```yml
spring:
  application:
    name: hello-spring-cloud-config
  cloud:
    config:
      label: master
      server:
        git:
          uri: https://github.com/topsale/spring-cloud-config
          search-paths: respo
          username:
          password:

server:
  port: 8888

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

相关配置说明，如下：

- `spring.cloud.config.label`：配置仓库的分支
- `spring.cloud.config.server.git.uri`：配置 Git 仓库地址（GitHub、GitLab、码云 ...）
- `spring.cloud.config.server.git.search-paths`：配置仓库路径（存放配置文件的目录）
- `spring.cloud.config.server.git.username`：访问 Git 仓库的账号
- `spring.cloud.config.server.git.password`：访问 Git 仓库的密码

注意事项：

- 如果使用 GitLab 作为仓库的话，`git.uri` 需要在结尾加上 `.git`，GitHub 则不用

### 附：HTTP 请求地址和资源文件映射

- `http://ip:port/{application}/{profile}[/{label}]`
- `http://ip:port/{application}-{profile}.yml`
- `http://ip:port/{label}/{application}-{profile}.yml`
- `http://ip:port/{application}-{profile}.properties`
- `http://ip:port/{label}/{application}-{profile}.properties`

## 2.4 测试

浏览器端访问：`http://localhost:8888/config-client/dev/master `显示如下：

```xml
<Environment> 
  <name>config-client</name>  
  <profiles> 
    <profiles>dev</profiles> 
  </profiles>  
  <label>master</label>  
  <version>9646007f931753d7e96a6dcc9ae34838897a91df</version>  
  <state/>  
  <propertySources> 
    <propertySources> 
      <name>https://github.com/test/spring-cloud-config/respo/config-client-dev.yml</name>  
      <source> 
        <foo>foo version 1</foo>  
        <demo.message>Hello Spring Config</demo.message> 
      </source> 
    </propertySources> 
  </propertySources> 
</Environment>
```

证明配置服务中心可以从远程程序获取配置信息

## 3. 搭建分布式配置中心客户端

### 3.1 pom.xml

其他项目就可以作为客户端，在pom.xml中增加 `spring-cloud-starter-config` 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### 3.2 application.yml

增加 Config Client 相关配置，并设置端口号为：`8889`

```yml
spring:
  application:
    name: hello-spring-cloud-config-client
  cloud:
    config:
      uri: http://localhost:8888
      name: config-client
      label: master
      profile: dev

server:
  port: 8889

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

相关配置说明，如下：

- `spring.cloud.config.uri`：配置服务中心的网址

- `spring.cloud.config.name`：配置文件名称的前缀

- `spring.cloud.config.label`：配置仓库的分支

- `spring.cloud.config.profile`

  ：配置文件的环境标识

  - dev：表示开发环境
  - test：表示测试环境
  - prod：表示生产环境

注意事项：

- 配置服务器的默认端口为 `8888`，如果修改了默认端口，则客户端项目就不能在 `application.yml` 或 `application.properties` 中配置 `spring.cloud.config.uri`，必须在 `bootstrap.yml` 或是 `bootstrap.properties` 中配置，原因是 `bootstrap` 开头的配置文件会被优先加载和配置，切记。

### 附：开启 Spring Boot Profile

我们在做项目开发的时候，生产环境和测试环境的一些配置可能会不一样，有时候一些功能也可能会不一样，所以我们可能会在上线的时候手工修改这些配置信息。但是 Spring 中为我们提供了 Profile 这个功能。我们只需要在启动的时候添加一个虚拟机参数，激活自己环境所要用的 Profile 就可以了。

操作起来很简单，只需要为不同的环境编写专门的配置文件，如：`application-dev.yml`、`application-prod.yml`， 启动项目时只需要增加一个命令参数 `--spring.profiles.active=环境配置` 即可，启动命令如下：

```bash
java -jar hello-spring-cloud-web-admin-feign-1.0.0-SNAPSHOT.jar --spring.profiles.active=pro
```

## 4. 小结

本章记录了`SpringCloudConfig`分布式配置中心的搭建及其具体使用过程与`Spring Boot Profile`多环境配置。

