---
title: Spring Cloud之Bus消息总线
date: 2022-06-28 10:55:46
categories: 
    - 微服务
    - Spring Cloud
cover: /img/post_cover/spring-cloud-bus.jpg
tags:
    - 微服务
    - Spring Cloud
    - 消息总线
---
# Spring Cloud之Bus消息总线

## 1. 消息代理

消息代理(`Message Broker`)是一种消息验证、传输、路由的架构模式。它在应用程序之间起到通信调度并最小化应用之间的依赖的作用，使得应用程序可以高效地解耦通信过程。消息代理是一个中间件产品，它的核心是一个消息的路由程序，用来实现接收和分发消息， 并根据设定好的消息处理流来转发给正确的应用。 它包括独立的通信和消息传递协议，能够实现组织内部和组织间的网络通信。设计代理的目的就是为了能够从应用程序中传入消息，并执行一些特别的操作，下面这些是在企业应用中，我们经常需要使用消息代理的场景：

- 将消息路由到一个或多个目的地。
- 消息转化为其他的表现方式。
- 执行消息的聚集、消息的分解，并将结果发送到它们的目的地，然后重新组合响应返回给消息用户。
- 调用`Web`服务来检索数据。
- 响应事件或错误。
- 使用`发布－订阅`模式来提供内容或基千主题的消息路由。

## 2. 什么是Spring Cloud Bus

Spring Cloud Bus 使用轻量级消息代理链接分布式系统的节点。可以通过消息代理广播**配置文件的更改**，或**服务之间的通讯**，也可以**用于监控**。解决了微服务数据变更，及时同步的问题。

## 3. 什么时候选用Spring Cloud Bus

微服务一般都做了集群方式部署，很多台服务器，而且在高并发下经常发生服务的扩容、缩容、上线、下线。这样，后端服务器的数量、`IP`就会变来变去，如果我们想进行一些线上的管理和维护工作，就需要维护服务器的`IP`。

比如我们需要更新配置、比如我们需要同时失效所有服务器上的某个缓存，都需要向所有的相关服务器发送命令，也就是调用一个接口。

你可能会说，我们一般会采用`zookeeper`的方式，统一存储服务器的`IP`地址，需要的时候，向对应服务器发送命令。这是一个方案，但是他的解耦性、灵活性、实时性相比消息总线都差那么一点。

总的来说，就是在我们需要把一个操作散发到所有后端相关服务器的时候，就可以选择使用 `Spring Cloud Bus` 了。

## 4. 环境准备

> 消息队列

- `RabbitMQ`

```shell
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

> 注册中心

- `spring-clud-demo-eureka-server01`和`spring-cloud-demo-eureka-server02`

> 配置中心

- `spring-cloud-demo-config-server`和`spring-cloud-demo-config-server01`

> 订单服务（配置中心客户端）

- `spring-cloud-demo-config-order`

## 5. Spring Cloud Bus实现配置文件刷新

### 客户端发起通知

消息总线（`Bus`）的典型应用场景就是**配置中心客户端刷新**。

`Spring Cloud Config`配置中心中基于`Actuator`的配置刷新，在只有一个配置中心客户端的时候，可以使用`Webhook`，设置手动刷新也比较方便，但是生产环境中客户端大多数都是集群部署，逐个手动刷新过于复杂，所以这种方案就不太适合。使用`Spring-Cloud-Bus`则完美的解决了这个问题。

借助 `Spring Cloud Bus` 的广播功能，让配置中心客户端都订阅配置更新事件，当配置更新时，触发其中一个端的更新事件，`Spring Cloud Bus` 就把此事件广播到其他订阅客户端，以此来达到批量更新。

![](客户端发起通知-1.png)

1. `Webhook` 监听被触发，给 `ConfigClient A` 发送 `bus-refresh` 请求刷新配置
2. `ConfigClient A` 读取 `ConfigServer` 中的配置，并且发送消息给 `Bus`
3. `Bus` 接收消息后广播通知其他 `Config Client`
4. 其他 `Config Client` 收到消息重新读取最新配置

> [Spring Cloud Bus](https://docs.spring.io/spring-cloud-bus/docs/2.2.4.RELEASE/reference/html/#quick-start)

#### 添加依赖

配置中心客户端添加依赖。

```xml
<!-- spring cloud starter bus amqp 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```



#### 配置文件

配置文件需要配置 `消息队列` 和 `bus-refresh` 自动刷新端点。`/actuator/bus-refresh` 端点会清除 `@RefreshScope` 缓存重新绑定属性。

配置中心客户端的`bootstrap.yml`核心配置。

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
      # 安全认证
      username: user
      password: 123456

  # 消息队列
  rabbitmq:
    host: 192.168.56.56
    port: 5672
    username: guest
    password: guest
    virtual-host: /

# 度量指标监控与健康检查
management:
  endpoints:
    web:
      base-path: /actuator   # 访问端点根路径，默认为 /actuator
      exposure:
        include: bus-refresh # 需要开启的端点
        #include: '*'          # 需要开启的端点，这里主要用到的是 refresh 这个端点
        #exclude:             # 不需要开启的端点

# 配置 Eureka Server 注册中心 Config默认从eureka 8671端口获取配置服务端的信息，如果不是8671，所以需要覆盖下注册中心的默认配置信息
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

```



#### 测试

##### 查看端点

访问：http://localhost:9091/actuator 可以看到已经开启了 `bus-refresh` 自动刷新端点。

![](客户端发起通知-2.png)

访问：http://localhost:9091/name，结果如下：

![](客户端发起通知-3.png)

##### 修改Git仓库配置

修改远程仓库的配置信息。

![](客户端发起通知-4.png)

##### 自动刷新

刷新页面发现结果并未改变。

![](客户端发起通知-5.png)

通过 Post 方式调用**任意客户端**的自动刷新端点：http://localhost:9091/actuator/bus-refresh 再次访问结果如下：

![](客户端发起通知-6.png)

##### 查看队列

`RabbitMQ`的控制台多了一个`springCloudBus`的交换机。

![](客户端发起通知-7.png)



该交换机下绑定了队列对应我们的配置中心客户端。

![](客户端发起通知-8.png)

#### 客户端发起通知缺陷

- 打破了微服务的职责单一性。微服务本身是业务模块，它本不应该承担配置刷新的职责。
- 破坏了微服务各节点的对等性。
- 存在一定的局限性。例如，微服务在迁移时，它的网络地址常常会发生变化，此时如果想要做到自动刷新，就不得不修改Webhook 的配置。

### 服务端发起通知

为了解决客户端发起通知缺陷，改用服务端发起通知。

![](服务端发起通知-1.png)

1. `Webhook`监听被触发，给 `Config Server` 发送 `bus-refresh` 请求刷新配置
2. `Config Server` 发送消息给 `Bus`
3. `Bus` 接收消息后广播通知所有 `Config Client` 
4. 各 `Config Client` 收到消息重新读取最新配置

#### 添加依赖

配置中心服务端添加`spring cloud starter bus amqp` 依赖。

```xml
<!-- spring cloud starter bus amqp 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

#### 配置文件

配置文件需要配置`消息队列`和`bus-refresh`自动刷新端点。`/actuator/bus-refresh` 端点会清除 `@RefreshScope` 缓存重新绑定属性。

配置中心服务端的`application.yml`核心配置。

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
  # 安全认证
  security:
    user:
      name: user
      password: 123456

  # 消息队列
  rabbitmq:
    host: 192.168.56.56
    port: 5672
    username: guest
    password: guest
    virtual-host: /

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

# 度量指标监控与健康检查
management:
  endpoints:
    web:
      base-path: /actuator    # 访问端点根路径，默认为 /actuator
      exposure:
        include: bus-refresh  # 需要开启的端点
        #exclude:             # 不需要开启的端点
```

#### 测试

##### 查看端点

访问：http://localhost:8888/actuator 和http://localhost:8889/actuator 可以看到已经开启了 `bus-refresh` 自动刷新端点。

![](服务端发起通知-2.png)



![](服务端发起通知-3.png)

访问：http://localhost:9091/name 结果如下：

![](服务端发起通知-4.png)

##### 修改Git配置

修改Git长裤配置信息 如下：

![](服务端发起通知-5.png)

##### 自动刷新

刷新页面发现结果并未改变。

通过 Post 方式调用**任意服务端**的自动刷新端点：http://localhost:8888/actuator/bus-refresh 再次访问结果如下：

![](服务端发起通知-6.png)

##### 查看队列

再来观察一下消息队列的 `UI` 界面，发现多了一个 `springCloudBus` 的交换机。

![](服务端发起通知-7.png)

### 局部刷新

假设有这样一种场景，我们开发了一个新的功能，此时需要对该功能进行测试。我们只希望其中一个微服务的配置被更新，等功能测试完毕，正式部署线上时再更新至整个集群。但是由于所有微服务都受 `Spring Cloud Bus` 的控制，我们更新了其中一个微服务的配置，就会导致其他服务也被通知去更新配置。这时候局部刷新的作用就体现出来了。

##### 刷新指定服务

修改 Git 仓库配置信息如下：

```yaml
# 自定义配置
name: order-service-prod-3.0
```

通过 `POST`方式调用**任意服务端**的自动刷新端点：http://localhost:8888/actuator/bus-refresh/order-service:9091 再次访问结果如下：

`9091` 端口的客户端已经更新配置。

![](局部刷新-1.png)

##### 刷新指定集群

假设现在功能测试完毕，需要正式部署线上更新至整个集群。但是由于 `Spring Cloud Bus` 控制着多个微服务集群（订单微服务、商品微服务等），而我们只想更新指定集群下的配置，这个时候就可以使用 `Bus` 提供的通配符更新方案。

修改 Git 仓库配置信息如下:

```yaml
# 自定义配置
name: order-service-prod-4.0
```

通过 Post 方式调用**任意服务端**的自动刷新端点：`http://localhost:8888/actuator/bus-refresh/order-service:**` 再次访问结果如下：

![](局部刷新-2.png)

至此 Bus 消息总线所有的知识点就学习结束了。



---

参考资料：

[Spring Cloud构建微服务架构（七）消息总线 - duanxz - 博客园 (cnblogs.com)](https://www.cnblogs.com/duanxz/p/6678545.html)

[通过消息总线Spring Cloud Bus实现配置文件刷新（使用Kafka或RocketMQ） - duanxz - 博客园 (cnblogs.com)](https://www.cnblogs.com/duanxz/p/11928587.html)

[Spring Cloud 系列之 Bus 消息总线 - 哈喽沃德先生 (mrhelloworld.com)](https://mrhelloworld.com/bus/)



