---
title: Spring Cloud之Config配置中心
date: 2022-06-25 17:58:43
categories: 
    - 微服务
    - Spring Cloud
cover: /img/post_cover/spring-cloud-config.jpg
tags:
    - 微服务
    - Spring Cloud
    - 配置中心
---
# Spring Cloud之Config配置中心

## 1. 服务配置现状
随着微服务系统的不断迭代，整个我服务就成为了一个**网状结构**，这个时候就要考虑整个微服务系统的**扩展性**、**伸缩性**、**耦合性**等等。其中一个重要的环节就是**配置管理**的问题。
## 2. 常规配置管理解决方案缺点

- 硬编码（需要修改代码、繁琐、风险大）
- 配置文件`properties`或者`yml`(集群环境下需要替换和重启)
- xml(需要打包和重启)

## 3. 为什么使用Spring Cloud Config
![](1625015953411.jpg)
针对于常规配置的缺点，所以采用`Spirng Cloud Config`**集中式**管理**每个服务**的配置信息。
`Spring Cloud Config`在微服务分布式系统中，采用`Server`服务端和`Client`客户端的方式来提供可扩展的配置服务。

- 服务端提供**配置文件的存储**，以**接口的形式将配置文件的内容**提供出去。
- 客户端通过接口获取数据，并依据此数据初始化自己的应用。

**配置中心负责管理所有服务的各种环境配置文件**。
## 4.Spring Cloud Config解决来了什么问题？
`Spring Cloud Config`解决了微服务配置的**中心化**、**版本控制**、**平台独立**、**语言独立**等问题。其特性如下：
- 提供了服务端和客户端支持
- 集中式管理分布式环境下的应用部署
- 属性值的加密和解密（对称加密和非对称加密）
- 基于 `Spring`环境，无缝与`Spring`应用集成
- 可用于任何语言开发程序
- 默认实现基于`Git`，可以进行版本管理
## 5. 项目环境
### 5.1 配置文件
![](1625126781619.png)
> 配置文件的名称是有规则的。例如`config-client-dev.yml`和`config-client-prod.yml`是同一个项目的不同环境。项目名称为`config-client`，`dev`表示测试环境，`prod`表示正式环境，`test`表示测试环境。
 `config-client.yml`

```yaml
server:
    port: 7777 # 端口
spring:
  application:
    name: config-client # 服务名称
# 自定义配置
name: config-client-default
```
 `config-client-dev.yml`
 ```yaml
 server:
  port: 7778 # 端口

spring:
  application:
    name: config-client # 服务名称
# 自定义配置
name: config-client-dev
 ```
 `config-client-prod.yml`
  ```yaml
  server:
  port: 7780 # 端口

spring:
  application:
    name: config-client # 服务名称
# 自定义配置
name: config-client-prod
  ```
 `config-client-test.yml`
  ```yaml
  server:
  port: 7779 # 端口

spring:
  application:
    name: config-client # 服务名称
# 自定义配置
name: config-client-test
  ```
 ## 6. 入门案例
### 6.1 创建服务端
参见`spring-cloud-demo`中的`spring-cloud-demo-config-server`模块。
- 访问规则
```yaml
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```
`{application}` : 应用名称（目标服务名称）
`{profile}`: 获取制定环境的配置。项目有测试环境，开发环境，正式环境。对应到配置文件就是application-{profile}.yml加以区分。默认值是default。
`{lable}`: 表示`git`分支，默认是`master`分支。
- 测试
http://localhost:8888/config-client/default
http://localhost:8888/config-client-dev.yml
http://localhost:8888/master/config-client-prod.yml
http://localhost:8888/master/config-client-test.properties

### 6.2 创建客户端
参见`spring-cloud-demo`中的`spring-cloud-demo-config-client`模块。

- 添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
    <version>2.2.8.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

- 添加配置文件

> 客户端的配置文件名称必须是`bootstrap.yml`
```yaml
spring:
  cloud:
    config:
      name: config-client # 配置文件名称，对应 git 仓库中配置文件前半部分
      uri: http://localhost:8888 # config-server 服务端地址
      label: master # git 分支
      profile: default # 指定环境
```

- 创建启动类

```java
package com.springcloud.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @Author: XiaoJie Wang
 * @Description:
 * @Date: Create in 14:54 2021/7/1
 */
@SpringBootApplication
public class SpringCloudConfigClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigClientApplication.class,args);
    }
}
```

- 创建测试控制层

```java
package com.springcloud.demo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @Author: XiaoJie Wang
 * @Description:
 * @Date: Create in 15:11 2021/7/1
 */
@RestController
public class ConfigController {
    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }
}
```

- 测试

`default`: [http://localhost:7777/name](http://localhost:7777/name)
`dev`: [http://localhost:7778/name](http://localhost:7778/name)
`test`: [http://localhost:7779/name](http://localhost:7779/name)
`prod`: [http://localhost:7780/name](http://localhost:7780/name)

## 7. Spring Cloud Config 高可用

如果我们的项目中使用了 Eureka 作为服务注册发现中心，那么 Spring Cloud Config 也应该注册到 Eureka，方便其他服务使用，并且可以注册多个配置中心服务端，实现高可用。

### 添加配置文件

在远程Git仓库增加配置文件

`order-service-dev.yml`

```yaml
server:
  port: 9090 # 端口

spring:
  application:
    name: order-service # 应用名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
      
# 自定义配置
name: order-service-dev

```

`order-service-prod.yml`

```yaml
server:
  port: 9091 # 端口

spring:
  application:
    name: order-service # 应用名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

# 自定义配置
name: order-service-prod

```

### 整合注册中心

#### 注册中心的依赖

`spring-cloud-demo-eureka-server01`和`spring-cloud-demo-eureka-server02`依赖一致

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springcloud-demo</artifactId>
        <groupId>com.springcloud.demo</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>spring-cloud-demo-eureka-server01</artifactId>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-eureka-server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
        <!-- spring boot security 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

    </dependencies>
</project>
```

#### 配置文件

`spring-cloud-demo-eureka-server01`的`application.yml`

```yaml
server:
  port: 8762 # 端口

spring:
  application:
    name: eureka-server # 应用名称(集群下相同)
  # 安全认证
  security:
    user:
      name: root
      password: 123456

# 配置Eureka Server 注册中心
eureka:
  instance:
    hostname: eureka01  # 主机名 不配置的时候根据操作系统的主机名来获取
    prefer-ip-address: true # 是否使用ip地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    registerWithEureka: true  # 是否将自己注册到注册中心，默认是true
    fetchRegistry: true       # 是否从注册中心获取服务配置信息，默认是true
    # 设置服务注册中心地址，指向另一个注册中心
    serviceUrl:                # 注册中心对外暴露的注册地址
      defaultZone: http://root:123456@localhost:8763/eureka/
  server:
    enable-self-preservation: false # true：开启自我保护模式，false：关闭自我保护模式
    eviction-interval-timer-in-ms: 60000 # 清理间隔（单位：毫秒，默认是 60*1000）
```

`spring-cloud-demo-eureka-server02`的`application.yml`

```yaml
server:
  port: 8763 # 端口

spring:
  application:
    name: eureka-server # 应用名称(集群下相同)
  # 安全认证
  security:
    user:
      name: root
      password: 123456

# 配置Eureka Server 注册中心
eureka:
  instance:
    hostname: eureka02 # 主机名 不配置的时候根据操作系统的主机名来获取
    prefer-ip-address: true # 是否使用ip地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    registerWithEureka: true  # 是否将自己注册到注册中心，默认是true
    fetchRegistry: true       # 是否从注册中心获取服务配置信息，默认是true
    # 设置服务注册中心地址，指向另一个注册中心
    serviceUrl:                # 注册中心对外暴露的注册地址
      defaultZone: http://root:123456@localhost:8762/eureka/
  server:
    enable-self-preservation: false # true：开启自我保护模式，false：关闭自我保护模式
    eviction-interval-timer-in-ms: 60000 # 清理间隔（单位：毫秒，默认是 60*1000）
```

#### 启动类

`spring-cloud-demo-eureka-server01`和`spring-cloud-demo-eureka-server02`启动类一致。

```java
package com.springcloud.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

/**
 * @Author: XiaoJie Wang
 * @Description:
 * @Date: Create in 23:22 2021/7/5
 */
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class,args);
    }
}
```

#### 注册中心配置类

```java
package com.springcloud.demo.config;

import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

/**
 * @Author: XiaoJie Wang
 * @Description:
 * @Date: Create in 22:15 2021/7/13
 */

@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http); // 加这句是为了访问 eureka 控制台和 /actuator 时能做安全控制
        http.csrf().ignoringAntMatchers("/eureka/**"); // 忽略 /eureka/** 的所有请求
    }
}
```

### Spring Cloud Config服务端

服务端和入门案例的配置中心相比多了 Eureka 的配置，其他地方都是一样的。

完成`spring-cloud-demo-config-server`的服务端构建完成以后再复刻一个`spring-cloud-demo-config-server`实现高可用。

#### 依赖

`spring-cloud-demo-config-server`和`spring-cloud-demo-config-server01`的依赖一致。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springcloud-demo</artifactId>
        <groupId>com.springcloud.demo</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>spring-cloud-demo-config</artifactId>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
        <!-- netflix eureka client 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

    </dependencies>

</project>
```

#### 配置文件

`spring-cloud-demo-config-server`的`application.yml`

```yaml
server:
  port: 8888 # 端口
spring:
  application:
    name: config-server # 应用名称
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/I10veU/config-repo.git # 配置文件远程仓库地址
          #username:            # Github等用户名
          #password:            # Github等密码
          #default-label:       # 配置文件分支
          #search-paths:        # 配置文件所在根目录

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```



`spring-cloud-demo-config-server01`的`application.yml`

```yaml
server:
  port: 8889 # 端口
spring:
  application:
    name: config-server # 应用名称
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/I10veU/config-repo.git # 配置文件远程仓库地址
          #username:            # Github等用户名
          #password:            # Github等密码
          #default-label:       # 配置文件分支
          #search-paths:        # 配置文件所在根目录

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

#### 启动类

`spring-cloud-demo-config-server`和`spring-cloud-demo-config-server01`启动类一致

```java
package com.springcloud.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * @Author: XiaoJie Wang
 * @Description:
 * @Date: Create in 15:11 2021/6/30
 */
@SpringBootApplication
@EnableConfigServer // 配置中心服务端注解
public class SpringCloudConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudConfigServerApplication.class, args);
    }
}
```



### Spring Cloud Config客户端

客户端加入 Eureka 以后，就不用直接和配置中心服务端打交道了，而是通过 Eureka 来访问。

#### 核心依赖

`spring-cloud-demo-config-order`的`pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springcloud-demo</artifactId>
        <groupId>com.springcloud.demo</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>spring-cloud-demo-config-order</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- netflix eureka client 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!-- spring cloud starter config 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
</project>
```

#### 配置文件

`order-service` 的 `bootstrap.yml`，Eureka 注册中心的配置在远程 Git 仓库的 `order-service-dev.yml` 中。

```yaml
spring:
  cloud:
    config:
      name: order-service # 配置文件名称，对应 git 仓库中配置文件前半部分
      label: master # git 分支
      profile: dev # 指定环境
      discovery:
        enabled: true # 开启
        service-id: config-server # 指定配置中心服务端的 service-id
```

#### 控制层

添加一个 `ConfigController` 用于测试获取配置文件信息。

```java
package com.springcloud.demo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ConfigController {

    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }

}
```

#### 启动类

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

#### 测试

启动注册中心：`spring-cloud-demo-eureka-server01`和`spring-cloud-demo-eureka-server02`

启动配置中心服务端：`spring-cloud-demo-config-server`和`spring-cloud-demo-config-server01`

启动配置中心客户端：`spring-cloud-demo-config-order`

当前环境在 `Eureka UI` 界面中如下：

![](高可用测试.png)

访问：http://localhost:9090/name 结果如下：

![](高可用测试-1.png)

## 8. 配置中心工作原理

![](配置中心工作原理.jpg)

开发人员将配置文件存储至 Git 远程仓库，或后期对 Git 远程仓库的文件进行修改。如果远程仓库发生了版本改变，Config Server 会将 Git 远程仓库中的文件同步至本地仓库中。大家仔细观察 Config Server 的控制台可以看到类似如下信息。

![](配置中心工作原理-1.png)

根据控制台信息打开对应的本地目录，会发现 Git 远程仓库中的文件已同步至本地仓库中。

![](配置中心工作原理-2.png)

> 为什么要这么做呢？因为我们要考虑网络波动的情况下，无法访问远程仓库的问题。

## 9. 配置中心自动刷新

### 问题演示

Spring Cloud Config在项目启动时才会加载配置文件。这样在修改配置文件内容后，不会自动刷新。例如，我们上面的演示，当服务已经启动的时候，修改Gitee上的配置内容，再次刷新页面，依然显示的时旧的配置信息，新的配置信息不会主动刷新。

![](配置中心自动刷新.png)

访问[http://localhost:9090/name](http://localhost:9090/name)，依然显示之前的配置信息

![](配置中心自动刷新-1.png)

重启 Config Client 以后，访问：http://localhost:9090/name 结果如下：

![](配置中心自动刷新-2.png)

但是，总不能每次修改了配置后重启服务吧。

提供了一个刷新机制，但是需要我们主动触发。那就是 `@RefreshScope` 注解并结合 `Actuator`，注意要引入 `spring-boot-starter-actuator`。

### 添加依赖

`spring-cloud-demo-config-order`中添加`spring-boot-starter-actuator` 依赖。

```xml
<!-- spring boot actuator 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 配置文件

其实这里主要用到的是 refresh 这个端点。以下为 `spring-cloud-demo-config-order` 的 `bootstrap.yml`

```yaml
spring:
  cloud:
    config:
      name: order-service # 配置文件名称，对应 git 仓库中配置文件前半部分
      label: master # git 分支
      profile: dev # 指定环境
      discovery:
        enabled: true # 开启
        service-id: config-server # 指定配置中心服务端的 service-id

spring:
  cloud:
    config:
      name: order-service # 配置文件名称，对应 git 仓库中配置文件前半部分
      uri: http://localhost:8888 # config-server 服务端地址
      label: master # git 分支
      profile: dev # 指定环境
      discovery:
        enabled: true # 开启
        service-id: config-server # 指定配置中心服务端的 service-id

# 度量指标监控与健康检查
management:
  endpoints:
    web:
      base-path: /actuator    # 访问端点根路径，默认为 /actuator
      exposure:
        include: '*'          # 需要开启的端点，这里主要用到的是 refresh 这个端点
        #exclude:             # 不需要开启的端点
```

### 控制层

在需要读取配置的类上增加 `@RefreshScope` 注解。

```java
![配置中心自动刷新-3](Spring Cloud Config配置中心/配置中心自动刷新-3.png)package com.springcloud.demo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@RestController
public class ConfigController {

    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }

}
```

### 测试

重启 `spring-cloud-demo-config-order`，访问：http://localhost:9090/actuator 可以看到 `refresh` 端点已开启。

![](配置中心自动刷新-3.png)

修改`Gitee`上的配置文件内容并提交，访问：http://localhost:9090/name，没有反应。

![](配置中心自动刷新-4.png)

接下来，我们发送 POST 请求到 http://localhost:9090/actuator/refresh 这个接口，用 Postman 之类的工具即可。

![](配置中心自动刷新-5.png)

再次访问：http://localhost:9090/name 结果如下：

![](配置中心自动刷新-6.png)

还有一种办法就是使用 `spring-cloud-config-monitor`，然后调用 `/monitor` 接口完成动态刷新。

### 在Gitee中配置Webhooks

自动刷新不能每次修改配置后，就用就用 Postman 访问一下 refresh 接口，这肯定不是自动刷新。Gitee提供了一种Webhook的方式，当有代码变更的时候，会调用我们设置的地址，来实现我们想达到的目的。

![](配置自动刷新-7.png)

填写回调的地址，也就是上面提到的`actuator/refresh` 这个地址，但是必须保证这个地址是可以被 Gitee 访问的。如果是内网就没办法了。一般公司内的项目都会有自己的代码管理工具，例如自建的Gitlab，Gitlab 也有 Webhook 的功能，这样就可以调用到内网的地址了。

![](配置自动刷新-8.png)

还有一种办法就是使用 `spring-cloud-config-monitor`，然后调用 `/monitor` 接口完成动态刷新。

> 参见官方文档，[Spring Cloud Config push notifications_and_spring_cloud_bus ](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_push_notifications_and_spring_cloud_bus)

## 10. Spring Cloud Bus 自动刷新

![](Spring-Cloud-Bus-自动刷新.png)

如果只有一个 `Spring Config Client` 的话，通过`Webhook`的方式还能勉强接受，但是如果客户端比较多的情况下，一个一个去手动刷新未免有点复杂。我们可以借助 `Spring Cloud Bus` 的广播功能，让 `Config Client` 都订阅配置更新事件，当配置更新时，触发其中一个端的更新事件，`Spring Cloud Bus` 就把此事件广播到其他订阅客户端，以此来达到批量更新。

> [Spring Cloud Bus](https://docs.spring.io/spring-cloud-bus/docs/2.2.4.RELEASE/reference/html/)

## 11. 配置中心加解密

考虑这样一个问题：所有的配置文件都存储在 Git 远程仓库，配置文件中的一些信息（比如数据库的用户名及密码）又是比较敏感的。所以，我们需要对这些敏感信息进行加密处理。主要的加密方法分为两种：一种是共享密钥加密（对称密钥加密），一种是公开密钥加密（非对称密钥加密）。

### 对称加解密 Symmetric encryption

> 对称加密是最快速、最简单的一种加密方式，加密（encryption）与解密（decryption）用的是同样的密钥（secret key）。

#### 检查加密环境

##### 版本问题

访问 Spring Cloud Config Server：http://localhost:8888/encrypt/status

- 检查结果1

`{"description":"No key was installed for encryption service","status":"NO_KEY"}` 

> 说明没有为加密服务安装密钥，也说明你使用的是较低的 `JDK` 版本。

**解决方案1**：

更换高版本 `JDK`，比如使用 `LTS` 版本 `JDK-11.0.6`。

**解决方案2**：

从 `Oracle` 官网下载对应 `JCE`，下载链接：https://www.oracle.com/java/technologies/javase-jce-all-downloads.html

![](配置中心加密-1.png)

> 图中红色框中内容已经足够说明原因：`JDK 9` 以及更高版本已附带策略文件，并在默认情况下启用。
>
> 如果你的当前环境必须使用低版本` JDK`，那么请下载对应 `JCE` 压缩包，下载解压后把 `local_policy.jar` 和 `US_export_policy.jar` 文件安装到需要安装 `JCE` 机器上的 `JDK` 或 `JRE` 的 `security` 目录下即可。

- 检查结果2

`{"description":"The encryption algorithm is not strong enough","status":"INVALID"}`

> 说明服务端未配置加密。

![](配置中心加密-2.png)

  

##### 配置问题

如果出现检查结果2所示的提示，说明服务端未配置加密。

`Spring Cloud Config Server`服务器创建配置文件，注意必须以`bootstrap.yml`命名，配置密钥信息即可。

```yaml
# 密钥
encrypt:
  key: demo
```

重启  Spring Cloud Config Server 访问：http://localhost:8888/encrypt/status 结果如下：

![](配置中心加密-3.png)



#### 加解密演示

##### 配置中心服务端

使用 `curl` 命令访问 `/encrypt` 端点对属性值 root 进行加密。反向操作 `/decrypt` 可解密。

```shell
curl http://localhost:8888/encrypt -d root
```

加密结果

```shell
Microsoft Windows [版本 10.0.19044.1766]
(c) Microsoft Corporation。保留所有权利。

C:\Users\wangxiaojie>curl http://localhost:8888/encrypt -d root
c9d615117d09a245164ed6ce938ee7de6ecdf98f5d55f15374f3537005304f1f
C:\Users\wangxiaojie>
```

解密

```shell
C:\Users\wangxiaojie>curl http://localhost:8888/decrypt -d c9d615117d09a245164ed6ce938ee7de6ecdf98f5d55f15374f3537005304f1f
root
C:\Users\wangxiaojie>
```

##### Git仓库

把加密后的数据更新到`Git`远程仓库的配置文件中。值得注意的是需要在加密结果前加`{cipher}`串，如果远程属性源包含加密的内容（以开头的值`{cipher}`），则将其解密，然后再通过HTTP发送给客户端。

![](配置中心加密-4.png)



##### 配置中心客户端

`spring-cloud-demo-config-order`控制层添加获取配置信息代码。

```java
package com.springcloud.demo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@RestController
public class ConfigController {

    @Value("${name}")
    private String name;

    @Value("${password}")
    private String password;

    @GetMapping("/name")
    public String getName() {
        return name;
    }

    @GetMapping("/password")
    public String getPassword() {
        return password;
    }


}
```

修改`spring-cloud-demo-config-order`配置文件`bootstrap.yml`，重启测试。

```yaml
spring:
  cloud:
    config:
      name: order-service # 配置文件名称，对应 git 仓库中配置文件前半部分
      uri: http://localhost:8888 # config-server 服务端地址
      label: master # git 分支
      profile: prod # 指定环境
      discovery:
        enabled: true # 开启
        service-id: config-server # 指定配置中心服务端的 service-id

# 度量指标监控与健康检查
management:
  endpoints:
    web:
      base-path: /actuator    # 访问端点根路径，默认为 /actuator
      exposure:
        include: '*'          # 需要开启的端点，这里主要用到的是 refresh 这个端点
        #exclude:             # 不需要开启的端点
```

访问：http://localhost:9091/password 返回解密后的结果。

![](配置中心加密-5.png)

### 非对称加解密 Asymmetric encryption

#### 对称加密和非对称加密的区别

对称加密算法在加密和解密时使用的是同一个密钥。只要拿到密钥，任何人都能破解。

![](对称加密.png)

非对称加密算法需要两个密钥来进行加密和解密，这两个密钥分别是公开密钥（public key 简称公钥）和私有密钥（private key 简称私钥）。在传输过程中，即使攻击者截获了传输的密文，并得到了公钥，也无法破解密文，因为使用专用密钥才能破解密文。

![](非对称加密.png)

> 图片来自《图解HTTP》

#### Java-keytool 使用说明

`Keytool` 用来管理私钥仓库(`keystore`)和与之相关的`X.509`证书链（用以验证与私钥对应的公钥），也可以用来管理其他信任实体。

默认大家都配置了 Java 的环境变量，打开 `CMD` 窗口运行以下命令。

```shell
# 生成名为 config.keystore 的 keystore 文件，别名为 config，加密算法类型使用 RSA，密钥库口令和密钥口令均为：config
keytool -genkeypair -keystore config.keystore -alias config -keyalg RSA -keypass config -storepass config
```

![](非对称加密-01.png)

此时在对应的目录下会生成一个 `config.keystore` 文件。

![](非对称加密-2.png)



#### 加解密演示

##### 配置中心服务端

将 `config.keystore` 文件添加至 `spring-cloud-demo-config-server01` 项目 `resources` 目录中。

![](非对称加密3.png)

创建 `bootstrap.yml` 添加非对称加解密配置。注意：值要跟 `CMD` 里输入的值对应不然会出错。

```yaml
# 非对称加解密
encrypt:
  key-store:
    location: classpath:config.keystore # keystore 文件存储路径
    alias: config # 密钥对别名
    password: config # storepass 密钥仓库
    secret: config # keypass 用来保护所生成密钥对中的私钥
```

`pom.xml` 添加避免 `maven` 过滤文件的配置。

```xml
<!-- build标签 常用于添加插件及编译配置 -->
<build>
    <!-- 读取配置文件 -->
    <resources>
        <resource>
            <directory>src/main/resources</directory>
        </resource>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
                <include>**/*.properties</include>
                <include>**/*.tld</include>
                <include>**/*.keystore</include>
            </includes>
            <filtering>false</filtering>
        </resource>
    </resources>
</build>
```

检查加密环境，访问：http://localhost:8889/encrypt/status 结果如下：

![](非对称加密-4.png)

使用 `curl` 命令访问 `/encrypt` 端点对属性值 root 进行加密。反向操作 `/decrypt` 可解密。

**加密**

```shell
curl http://localhost:8889/encrypt -d root
```

```shell
D:\>curl http://localhost:8889/encrypt -d root
AQB1Ty71sHBjQfLq7tGW/HoaL7k2Z+5arwCO0sWg7mQ/EQ7jpz1s1Ubak1gNcpIcxPcwUGAsYE0PKeV4JfD6yezD7D/LDq+q2BIJkMVqvbgh0LCcqPf7jc9s5Ky0hYB4NZqTlWW0BStFLw9EX2yC/OWfqCAUB65lWNc5KXCCXpHVSoBlXI+xkPLr1eYxo3rex/0KY/hF0cEZ/fmcgac9gEgjg5GGfiK2Z7/ug9r6bDnzW4CoOYbVHYYfzxfvqu0HKze5WCEZbG0xHUAUkOj3PnY3kQGoOTe9+AMLxtnTKOof3GAJX/XKLTHMmDjaMQzDWQEFxLhUTzNAzJ5anh5tINS0OsLnKUywJltkLouSPlICXgMx4XnKX91DSsFNzzyqBSg=
```

**解密**

```shell
curl http://localhost:8889/decrypt -d AQB1Ty71sHBjQfLq7tGW/HoaL7k2Z+5arwCO0sWg7mQ/EQ7jpz1s1Ubak1gNcpIcxPcwUGAsYE0PKeV4JfD6yezD7D/LDq+q2BIJkMVqvbgh0LCcqPf7jc9s5Ky0hYB4NZqTlWW0BStFLw9EX2yC/OWfqCAUB65lWNc5KXCCXpHVSoBlXI+xkPLr1eYxo3rex/0KY/hF0cEZ/fmcgac9gEgjg5GGfiK2Z7/ug9r6bDnzW4CoOYbVHYYfzxfvqu0HKze5WCEZbG0xHUAUkOj3PnY3kQGoOTe9+AMLxtnTKOof3GAJX/XKLTHMmDjaMQzDWQEFxLhUTzNAzJ5anh5tINS0OsLnKUywJltkLouSPlICXgMx4XnKX91DSsFNzzyqBSg=
```

```shell
D:\>curl http://localhost:8889/decrypt -d AQB1Ty71sHBjQfLq7tGW/HoaL7k2Z+5arwCO0sWg7mQ/EQ7jpz1s1Ubak1gNcpIcxPcwUGAsYE0PKeV4JfD6yezD7D/LDq+q2BIJkMVqvbgh0LCcqPf7jc9s5Ky0hYB4NZqTlWW0BStFLw9EX2yC/OWfqCAUB65lWNc5KXCCXpHVSoBlXI+xkPLr1eYxo3rex/0KY/hF0cEZ/fmcgac9gEgjg5GGfiK2Z7/ug9r6bDnzW4CoOYbVHYYfzxfvqu0HKze5WCEZbG0xHUAUkOj3PnY3kQGoOTe9+AMLxtnTKOof3GAJX/XKLTHMmDjaMQzDWQEFxLhUTzNAzJ5anh5tINS0OsLnKUywJltkLouSPlICXgMx4XnKX91DSsFNzzyqBSg=
root
D:\>
```



##### Git仓库

把加密后的数据更新到 Git 远程仓库的配置文件中。值得注意的是需要在加密结果前添加 `{cipher}` 串，如果远程属性源包含加密的内容（以开头的值`{cipher}`），则将其解密，然后再通过HTTP发送给客户端。

![](非对称加密-5.png)



##### 配置中心客户端

`spring-cloud-demo-config-order`配置文件`bootstrap.yml`

```yaml
spring:
  cloud:
    config:
      name: order-service # 配置文件名称，对应 git 仓库中配置文件前半部分
      label: master # git 分支
      profile: prod # 指定环境
      discovery:
        enabled: true # 开启
        service-id: config-server # 指定配置中心服务端的 service-id

# 度量指标监控与健康检查
management:
  endpoints:
    web:
      base-path: /actuator    # 访问端点根路径，默认为 /actuator
      exposure:
        include: '*'          # 需要开启的端点，这里主要用到的是 refresh 这个端点
        #exclude:             # 不需要开启的端点

```

访问：http://localhost:9091/password 返回解密后的结果。

![](非对称加密-6.png)



## 12. 配置中心用户安全认证

`Spring Cloud Config Server`服务器的配置文件之前访问是不受任何约束的，而且直接通过 `Config Server` 访问配置文件信息，加密的内容就会解密后直接显示在浏览器中，这样加密就没有意义。当然不是，我们只需要添加用户安全认证即可。

![](安全认证-1.png)

### 添加依赖

`spring-cloud-demo-config-server01`的`pom.xml`中添加依赖

```xml
<!-- spring boot security 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

```

### 配置文件

`spring-cloud-demo-config-server01`的`application.yml`

```yaml
spring:
  # 安全认证
  security:
    user:
      name: user
      password: 123456
```

`spring-cloud-demo-config-order`的`bootstrap.yml`添加安全认证配置。

```yaml
```





### 测试

#### 服务端

访问：http://localhost:8889/order-service-prod.yml 被重定向至登录页。

![](安全认证-2.png)

输入用户名和密码后，结果如下：

![](安全认证-3.png)

#### 客户端

访问：http://localhost:9091/password 结果如下：

![](安全认证-4.png)

Config 配置中心所有的知识点就学习结束了。



