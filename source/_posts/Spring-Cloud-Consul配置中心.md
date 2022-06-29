---
title: Spring Cloud Consul配置中心
date: 2022-06-29 09:07:16
categories: 
    - 微服务
    - Spring Cloud
cover: /img/post_cover/spring-cloud-consul.png
tags:
    - 微服务
    - Spring Cloud
    - 配置中心
---
# Spring Cloud Consul配置中心

之前学习过了`Spring Cloud Config`，它提供了配置中心的功能，但是需要配合`git`、`svn`或外部存储（例如各种数据库），而且需要配合`Spring Cloud Bus`实现配置刷新。

`Spring Cloud Consul`，之前学习了它作为注册中心的使用方案，且作为 `Spring Cloud` 官方推荐替换 `Eureka` 注册中心的方案。既然使用了 `Consul`，就可以使用 `Consul` 提供的配置中心功能，并且不需要额外的 `git` 、`svn`、数据库等配合，且无需配合 `Bus` 即可实现配置刷新。

> [Spring Cloud Consul](https://docs.spring.io/spring-cloud-consul/docs/2.2.7.RELEASE/reference/html/#spring-cloud-consul-config)

## 1. Consul 介绍

[Console](https://www.consul.io/)是 `HashiCorp` 公司推出的开源工具，用于实现分布式系统的服务发现与配置。与其它分布式服务注册与发现的方案，`Consul` 的方案更**一站式**，内置了服务注册与发现框架、分布式一致性协议实现、健康检查、`Key/Value 存储（配置中心）`、多数据中心方案，不再需要依赖其它工具（比如 `ZooKeeper` 等），使用起来也较为简单。

　　`Consul` 使用 `Go` 语言编写，因此具有天然可移植性（支持`Linux`、`Windows` 和 `Mac OS`）；安装包仅包含一个可执行文件，方便部署，与 `Docker` 等轻量级容器可无缝配合。

## 2. Consul 特性

- 服务发现
- 健康检查
- `KV`存储（配置中心）
- 安全服务通信
- 多数据中心

## 3. Consul 安装

### 下载

[Downloads | Consul by HashiCorp](https://www.consul.io/downloads)

### 安装

> Windows下单节点安装

下载后的压缩包解压后就只有一个`consul.exe`的执行文件。

`cd` 到对应的目录下，使用 `cmd` 启动 `Consul`。

```shell
# -dev表示开发模式运行
consul agent -dev -client=0.0.0.0
```

为了方便启动，也可以在 `consul.exe` 同级目录下创建一个脚本来启动，脚本内容如下：

```shell
consul agent -dev -client=0.0.0.0
pause
```

访问管理后台：http://localhost:8500/ 看到下图意味着我们的 Consul 服务启动成功了。

![](consul-配置中心-1.png)

## 4. 初始化配置

### 创建基本目录

使用 `Consul` 作为配置中心，第一步我们先创建目录，把配置信息存储至 `Consul`。

点击菜单 `Key/Value` 再点击 `Create` 按钮。

![](consul-配置中心-2.png)

创建 `config/` 基本目录，可以理解为配置文件所在的最外层文件夹。

![](consul-配置中心-3.png)



### 创建应用目录

- 点击 `config` 进入文件夹。

![](consul-配置中心-4.png)

- 再点击`Create`按钮。

![](consul-配置中心-5.png)

- 创建 `orderService/` 应用目录，存储对应微服务应用的 `default` 环境配置信息。

![](consul-配置中心-6.png)

  

### 多环境应用目录

假设我们的项目有多环境：`default`、`test`、`dev`、`prod`，在 `config` 目录下创建多环境目录。

- `orderService` 文件夹对应 `default` 环境
- `orderService-test` 文件夹对应 `test` 环境
- `orderService-dev` 文件夹对应 `dev` 环境
- `orderService-prod` 文件夹对应 `prod` 环境

![](consul-配置中心-7.png)



### 初始化配置

以 `dev`环境为例，点击`orderService-dev`进入文件夹。

![](consul-配置中心-8.png)

点击`Create`按钮准备创建`Key/Value`配置信息。

![](consul-配置中心-9.png)

填写`Key`：`orderServiceConfig`

填写`Value`:

```yaml
name: order-service-dev
mysql:
  host: localhost
  port: 3006
  username: root
  password: root
```

![](consul-配置中心-10.png)

假设以上内容为订单微服务的配置信息，下面我们通过案例来加载 `Consul` 配置中心中的配置信息。

## 5. 环境准备

- `spring-cloud-demo-consul-config` `Maven`聚合工程　
- 订单服务：`order-service01`和`order-service02`

## 6. 实践案例

### 添加依赖

需要从 Consul 获取配置信息的项目主要添加 `spring-cloud-starter-consul-config` 依赖，完整依赖如下：

`order-service01` 和 `order-service02` 核心依赖一致。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>spring-cloud-demo-consul-config</artifactId>
        <groupId>com.springcloud.demo</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>order-service01</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <!-- spring boot web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- spring boot actuator 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- spring cloud consul discovery 服务发现依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
        <!-- spring cloud consul config 配置中心依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-config</artifactId>
        </dependency>
    </dependencies>
</project>
```

　

### 配置文件

配置文件必须叫 `bootstrap.yml` 我们除了使用 `Consul` 配置中心功能之外，把微服务也注册到 `Consul` 注册中心去。`order-service01` 和 `order-service02` 的配置项中除了端口和注册实例 id 之外，其余配置项一致，完整配置如下：

```yaml
server:
  port: 9090 # 端口

spring:
  application:
    name: order-service # 应用名称
  profiles:
    active: dev # 指定环境，默认加载 default 环境
  cloud:
    consul:
      # Consul 服务器地址
      host: localhost
      port: 8500
      # 配置中心相关配置
      config:
        # 是否启用配置中心，默认值 true 开启
        enabled: true
        # 设置配置的基本文件夹，默认值 config 可以理解为配置文件所在的最外层文件夹
        prefix: config
        # 设置应用的文件夹名称，默认值 application 一般建议设置为微服务应用名称
        default-context: orderService
        # 配置环境分隔符，默认值 "," 和 default-context 配置项搭配
        # 例如应用 orderService 分别有环境 default、dev、test、prod
        # 只需在 config 文件夹下创建 orderService、orderService-dev、orderService-test、orderService-prod 文件夹即可
        profile-separator: '-'
        # 指定配置格式为 yaml
        format: YAML
        # Consul 的 Key/Values 中的 Key，Value 对应整个配置文件
        data-key: orderServiceConfig
        # 以上配置可以理解为：加载 config/orderService/ 文件夹下 Key 为 orderServiceConfig 的 Value 对应的配置信息
        watch:
          # 是否开启自动刷新，默认值 true 开启
          enabled: true
          # 刷新频率，单位：毫秒，默认值 1000
          delay: 1000
      # 服务发现相关配置
      discovery:
        register: true                                # 是否需要注册
        instance-id: ${spring.application.name}-01    # 注册实例 id（必须唯一）
        service-name: ${spring.application.name}      # 服务名称
        port: ${server.port}                          # 服务端口
        prefer-ip-address: true                       # 是否使用 ip 地址注册
        ip-address: ${spring.cloud.client.ip-address} # 服务请求 ip
```

### 配置文件实体类

`order-service01` 和 `order-service02` 实体类代码一致。

```java
package com.springcloud.demo.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@ConfigurationProperties(prefix = "mysql")
@Component
@Data
public class MySQLProperties {

    private String host;

    private Integer port;

    private String username;

    private String password;
}
```

### 控制层

`order-service01` 和 `order-service02` 控制层代码一致。

> ❗注意：需要添加 `@RefreshScope` 注解用于重新刷新作用域实现属性值自动刷新。

```java
package com.springcloud.demo.controller;

import com.springcloud.demo.config.MySQLProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@RestController
public class ConfigController {


    @Autowired
    private MySQLProperties mySQLProperties;

    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }

    @GetMapping("/mysql")
    public MySQLProperties getMySQLProperties() {
        return mySQLProperties;
    }
}
```

### 启动类

`order-service01` 和 `order-service02` 启动类代码一致。

```java
package com.springcloud.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### 测试

#### 修改配置信息前

访问：http://localhost:9090/name 结果如下：

![](consul-配置中心-11.png)

访问：http://localhost:9090/mysql 结果如下：

![](consul-配置中心-12.png)

#### 修改配置信息

修改 Consul 配置中心 `orderService-dev` 环境的配置信息为：

```yaml
name: order-service-dev-v2.2
mysql:
  host: localhost
  port: 3006
  username: root123456
  password: root123456
```

![](consul-配置中心-13.png)

#### 修改配置信息后

控制台打印信息如下：

```shell
2022-06-28 23:00:21.431  INFO 52920 --- [TaskScheduler-1] o.s.cloud.commons.util.InetUtils         : Cannot determine local hostname
2022-06-28 23:00:22.586  INFO 52920 --- [TaskScheduler-1] b.c.PropertySourceBootstrapConfiguration : Located property source: [BootstrapPropertySource {name='bootstrapProperties-config/order-service-dev/'}, BootstrapPropertySource {name='bootstrapProperties-config/order-service/'}, BootstrapPropertySource {name='bootstrapProperties-config/orderService-dev/'}, BootstrapPropertySource {name='bootstrapProperties-config/orderService/'}]
2022-06-28 23:00:22.587  INFO 52920 --- [TaskScheduler-1] o.s.boot.SpringApplication               : The following profiles are active: dev
2022-06-28 23:00:22.592  INFO 52920 --- [TaskScheduler-1] o.s.boot.SpringApplication               : Started application in 2.457 seconds (JVM running for 593.8)
2022-06-28 23:00:22.647  INFO 52920 --- [TaskScheduler-1] o.s.c.e.event.RefreshEventListener       : Refresh keys changed: [spring.cloud.client.hostname, name, mysql.password, mysql.username]
```

Consul 使用 Spring 定时任务 `Spring TaskScheduler`来监听配置文件的更新。

默认情况下，它是一个定时任务线程池 `ThreadPoolTaskScheduler`，其`poolSize`值为 `1`。要更改`TaskScheduler`，请创建一个`TaskScheduler` 使用 `ConsulConfigAutoConfiguration.CONFIG_WATCH_TASK_SCHEDULER_NAME` 常量命名的 bean 类型。

> [Spring Cloud Consul](https://docs.spring.io/spring-cloud-consul/docs/2.2.7.RELEASE/reference/html/#spring-cloud-consul-config-watch)

访问：http://localhost:9090/name 结果如下：

![](consul-配置中心-14.png)

访问：http://localhost:9090/mysql 结果如下：

![](consul-配置中心-15.png)





## 7. 总结

`HashiCorp` 公司的 `Consul` 可谓是一款全能组件。可用于提供服务发现和服务配置的工具。用 `go` 语言开发，具有很好的可移植性，被 `Spring Cloud` 纳入其中。

在**注册中心**方面，`Netflix Eureka` 停止新版本开发，`Consul` 成为了优秀的可替代方案。

在**配置中心**方面，`Consul` 亦可替代 `Spring Cloud Config` 作为配置中心使用，且无需配合 `Git`、`SVN` 等工具，无需配合 `Bus` 消息总线即可实现集群配置更新。



---

参考资料：

[Spring Cloud 系列之 Consul 配置中心 - 哈喽沃德先生 (mrhelloworld.com)](https://mrhelloworld.com/consul-config/)


