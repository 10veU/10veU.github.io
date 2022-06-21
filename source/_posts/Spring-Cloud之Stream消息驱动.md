---
title: Spring Cloud之Stream消息驱动
date: 2022-06-21 15:15:04
categories: 
    - 微服务
    - Spring Cloud
cover: /img/post_cover/spring-cloud-stream.jpg
tags:
    - 微服务
    - Spring Cloud
    - 消息驱动
---
# Spring Cloud之Stream消息驱动

在实际的企业开发中，消息中间件是至关重要的组件之一。消息中间件主要解决应用解耦，异步消
息，流量削锋等问题，实现高性能，高可用，可伸缩和最终一致性架构。

不同的中间件其实现方式，内部结构是不一样的。如常见的`RabbitMQ`和`Kafka`，由于这两个消息中间件的架构上的不同，像`RabbitMQ`有`exchange`，`kafka`有`Topic`，`partitions`分区，这些中间件的差异性导致我们实际项目开发给我们造成了一定的困扰，我们如果用了两个消息队列的其中一种，后面的业务需求，我想往另外一种消息队列进行迁移，这时候无疑就是一个灾难性的，一大堆东西都要重新推倒重新做，因为它跟我们的系统耦合了，这时候 `Springcloud Stream` 给我们提供了一种解耦合的方式。

## 1. 消息中间件的应用场景

### 应用解耦

假设公司有几个不同的系统，各系统在某些业务有联动关系，比如 A 系统完成了某些操作，需要触发 B 系统及 C 系统，但是各个系统之间产生了耦合。针对这种场景，用消息中间件就可以完成解耦，当 A 系统完成操作时将数据放进消息队列，B 和 C 系统去订阅消息就可以了，这样各系统只要约定好消息的格式就可以了。

- 传统模式

![](中间件应用场景-应用解耦-1.png)

- 中间件模式

![](中间件应用场景-应用解耦-2.png)

### 异步处理

比如用户在电商网站下单，下单完成后会给用户推送短信或邮件，发短信和邮件的过程就可以异步完成。因为下单付款才是核心业务，发邮件和短信并不属于核心功能，且可能耗时较长，所以针对这种业务场景可以选择先放到消息队列中，由其他服务来异步处理。

- 传统模式

![](中间件应用场景-异步处理-1.png)

- 中间件模式

![](中间件应用场景-异步处理-2png.png)

### 流量削峰

比如秒杀活动，一下子进来好多请求，有的服务可能承受不住瞬时高并发而崩溃，针对这种场景，在中间加一层消息队列，把请求先入队列，然后再把队列中的请求平滑的推送给服务，或者让服务去队列拉取。

- 传统模式

![](中间件应用场景-流量削峰-1.png)

- 中间件模式

![](中间件应用场景-流量削峰-2.png)

  

### 日志处理

对于小型项目来说，我们通常对日志的处理没有那么多的要求，但是当用户量，数据量达到一定的峰值之后，问题就会随之而来。比如：

- 用户日志怎么存放
- 用户日志存放后怎么利用
- 怎么在存储大量日志而不对系统造成影响

等很多其他的问题，这样我们就需要借助消息队列进行业务的上解耦，数据上更好的传输。

Kafka 最开始就是专门为了处理日志产生的。

### 总结

消息队列，是分布式系统中重要的组件，其通用的使用场景可以简单地描述为：**当不需要立即获得结果，但是并发量又需要进行控制的时候，差不多就是需要使用消息队列的时候**。在项目中，将一些无需即时返回且耗时的操作提取出来，进行了异步处理，而这种异步处理的方式大大的节省了服务器的请求响应时间，从而提高了系统的吞吐量。

当碰到上面的几种情况的时候，就要考虑用消息队列了。如果你碰巧使用的是 RabbitMQ 或者 Kafka ，而且同样也是在使用 Spring Cloud ，那可以考虑下用 Spring Cloud Stream。

## 2. 什么是Spring Cloud Stream

Spring Cloud Stream 为一些供应商的消息中间件产品（目前集成了 RabbitMQ 和 Kafka）提供了个性化的自动化配置实现，并且引入了发布/订阅、消费组以及消息分区这三个核心概念。简单地说，Spring Cloud Stream 本质上就是整合了 Spring Boot 和 Spring Integration, 实现了一套轻量级的消息驱动的微服务框架。

Spring Cloud Stream 解决了开发人员无感知的使用消息中间件的问题，因为 Stream 对消息中间件的进一步封装，可以做到代码层面对中间件的无感知，甚至于动态的切换中间件，使得微服务开发的高度解耦，服务可以关注更多自己的业务流程。

## 3. 核心概念

Spring Cloud Stream 提供了许多抽象概念和专业术语，可以简化消息驱动的微服务应用程序的编写。

#### 应用模型（`Application Model`）

应用程序通过 inputs 或者 outputs 来与 Spring Cloud Stream 中Binder 交互，通过我们配置来绑定，而 Spring Cloud Stream 的 Binder 负责与中间件交互。所以，我们只需要搞清楚如何与 Spring Cloud Stream 交互就可以方便使用消息驱动的方式。

![](SCSt-with-binder.png)



#### 抽象绑定器（The Binder Abstraction）

Spring Cloud Stream实现 Kafka 和 RabbitMQ 的 Binder 实现，也包括了一个TestSupportBinder，用于测试。你也可以写根据API去写自己的Binder.

Spring Cloud Stream 同样使用了Spring boot的自动配置，并且抽象的Binder使Spring Cloud Stream的应用获得更好的灵活性，比如：我们可以在`application.yml`或`application.properties`中指定参数进行配置使用Kafka或者RabbitMQ，而无需修改我们的代码。

通过`Binder` ，可以方便地连接中间件，可以通过修改`application.yml`中的`spring.cloud.stream.bindings.input.destination` 来进行改变消息中间件（对应于Kafka的topic，RabbitMQ的exchanges），在这两者间的切换甚至不需要修改一行代码。

#### 发布-订阅(Persistent Publish-Subscribe Support)

应用程序之间的通信遵循发布-订阅模型，其中数据通过共享主题进行广播。

![](核心概念-发布-订阅.png)

> 其中topic对应于Spring Cloud Stream中的destinations（Kafka 的topic，RabbitMQ的 exchanges）

#### 消费组(Consumer Groups)

虽然发布-订阅模型使得通过共享主题连接应用程序变得非常容易，但是通过创建给定应用程序的多个实例来扩大规模的能力同样重要。但是如果这些实例都去消费这条数据，那么很可能会出现重复消费的问题，我们只需要同一应用中只有一个实例消费该消息，这时我们可以通过消费组来解决这种应用场景， **当一个应用程序不同实例放置在一个具有竞争关系的消费组中，组里面的实例中只有一个能够消费消息**。

每个使用者绑定都可以使用 `spring.clod.stream.bindings.<bindingName>.group` 属性来指定组名。

如下图所示：

通过网络传递过来的消息通过主题，按照分组名进行传递到消费者组中

此时可以通过`spring.cloud.stream.bindings.input.group=Group-A`或`spring.cloud.stream.bindings.input.group=Group-B`进行指定消费组

![](核心概念-消费组.png)

所有订阅指定主题的组都会收到发布消息的一个备份，每个组中只有一个成员会收到该消息；如果没有指定组，那么默认会为该应用分配一个匿名消费者组，与所有其它组处于 订阅-发布 关系中。`ps:`也就是说如果管道没有指定消费组，那么这个匿名消费组会与其它组一起消费消息，出现了重复消费的问题。

##### 消费者类型（Consumer Types）

支持有两种消费者类型：

- Message-driven (消息驱动型，有时简称为`异步`)
- Polled (轮询型，有时简称为 `同步`)

在Spring Cloud 2.0版本前只支持 Message-driven这种异步类型的消费者，消息一旦可用就会传递，并且有一个线程可以处理它；当你想控制消息的处理速度时，可能需要用到同步消费者类型。

持久化

**一般来说所有拥有订阅主题的消费组都是持久化的，除了匿名消费组。** Binder的实现确保了所有订阅关系的消费订阅是持久的，一个消费组中至少有一个订阅了主题，那么被订阅主题的消息就会进入这个组中，无论组内是否停止。

> 注意： 匿名订阅本身是非持久化的，但是有一些Binder的实现（比如RabbitMQ）则可以创建非持久化的组订阅

通常情况下，当有一个应用绑定到目的地的时候，最好指定消费消费组。扩展Spring Cloud Stream应用程序时，必须为每个输入绑定指定一个使用者组。这样做可以防止应用程序的实例接收重复的消息（除非需要这种行为，这是不寻常的）。

##### 分区支持（Partitioning Support）

在消费组中我们可以保证消息不会被重复消费，但是在同组下有多个实例的时候，我们无法确定每次处理消息的是不是被同一消费者消费，分区的作用就是为了**确保具有共同特征标识的数据由同一个消费者实例进行处理**，当然前边的例子是狭义的，通信代理（broken topic）也可以被理解为进行了同样的分区划分。Spring Cloud Stream 的分区概念是抽象的，可以为不支持分区Binder实现（例如RabbitMQ）也可以使用分区。

![](核心概念-分区支持.png)

> 注意：要使用分区处理，你必须同时对生产者和消费者进行配置。

## 4. 编程模型（`Programming Model`）

要理解编程模型，您应该熟悉以下核心概念:

- **Destination Binders(**目的地绑定器)：负责提供与外部消息传递系统集成的组件。
- **Bindings（绑定）:**外部消息传递系统和应用程序之间的桥梁提供了消息的生产者和消费者(由目标绑定器创建)。
- **Message（消息）**： 用于生产者、消费者通过Destination Binders沟通的规范数据。

![](编程模型.png)

## 5. 工作原理

通过定义**绑定器**作为中间层，实现了应用程序与消息中间件细节之间的隔离。通过向应用程序暴露统一的 **Channel** 通道，使得应用程序不需要再考虑各种不同的消息中间件的实现。当需要升级消息中间件，或者是更换其他消息中间件产品时，我们需要做的就是更换对应的 `Binder` 绑定器而不需要修改任何应用逻辑。

![](工作原理.jpg)

该模型图中有如下几个核心概念：

- `Source`：当需要发送消息时，我们就需要通过 `Source.java`，它会把我们所要发送的消息进行序列化（默认转换成 JSON 格式字符串），然后将这些数据发送到 Channel 中；
- `Sink`：当我们需要监听消息时就需要通过 `Sink.java`，它负责从消息通道中获取消息，并将消息反序列化成消息对象，然后交给具体的消息监听处理；
- `Channel`：通常我们向消息中间件发送消息或者监听消息时需要指定主题（Topic）和消息队列名称，一旦我们需要变更主题的时候就需要修改消息发送或消息监听的代码。通过 `Channel` 对象，我们的业务代码只需要对应 `Channel` 就可以了，具体这个 Channel 对应的是哪个主题，可以在配置文件中来指定，这样当主题变更的时候我们就不用对代码做任何修改，从而实现了与具体消息中间件的解耦；
- `Binder`：通过不同的 `Binder` 可以实现与不同的消息中间件整合，`Binder` 提供统一的消息收发接口，从而使得我们可以根据实际需要部署不同的消息中间件，或者根据实际生产中所部署的消息中间件来调整我们的配置。

## 6. 环境准备


- 注册中心:  `spring-cloud-demo-eureka-server01`， `spring-cloud-demo-eureka-server02`
- Stream Demo模块：`spring-cloud-demo-stream`
- Rabbit MQ消息队列


```shell
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

  

## 7. 入门案例

### 消息生产者

在Stream Demo（`spring-cloud-demo-stream`）模块下创建子项目`stream-producer`。

#### 添加依赖

要使用 RabbitMQ 绑定器，可以通过使用以下 Maven 坐标将其添加到 Spring Cloud Stream 应用程序中：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

或者使用 Spring Cloud Stream RabbitMQ Starter：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

#### 配置文件

配置 RabbitMQ 消息队列和 Stream 消息发送与接收的通道。

```yaml
server:
  port: 8001 # 端口

spring:
  application:
    name: stream-producer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息发送通道
        # 与 org.springframework.cloud.stream.messaging.Source 中的 @Output("output") 注解的 value 相同
        output:
          destination: stream.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

```

#### 发送消息

`MessageProducer.java`

```java
package com.springcloud.demo.producer;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(Source.class)
public class MessageProducer {

    @Autowired
    private Source source;

    /**
     * 发送消息
     *
     * @param message
     */
    public void send(String message) {
        source.output().send(MessageBuilder.withPayload(message).build());
    }

}

```

#### 启动类

```java
package com.springcloud.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StreamProducerApplication {
    public static void main(String[] args) {
        SpringApplication.run(StreamProducerApplication.class, args);
    }
}
```

### 消息消费者

在Stream Demo（`spring-cloud-demo-stream`）模块下创建子项目`stream-consumer`。

#### 添加依赖

要使用 RabbitMQ 绑定器，可以通过使用以下 Maven 坐标将其添加到 Spring Cloud Stream 应用程序中：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

或者使用 Spring Cloud Stream RabbitMQ Starter：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

#### 配置文件

```yaml
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

#### 接收消息

`MessageConsumer.java`

```java
package com.springcloud.demo.consumer;

import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.stereotype.Component;

/**
 * 消息消费者
 */
@Component
@EnableBinding(Sink.class)
public class MessageConsumer {

    /**
     * 接收消息
     *
     * @param message
     */
    @StreamListener(Sink.INPUT)
    public void receive(String message) {
        System.out.println("message = " + message);
    }

}
```

#### 启动类

```java
package com.springcloud.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StreamApplication {
    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }
}
```

### 测试

#### 单元测试

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.StreamProducerApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(classes = {StreamProducerApplication.class})
public class MessageProducerTest {


    @Autowired
    private MessageProducer messageProducer;

    @Test
    public void testSend() {
        messageProducer.send("hello spring cloud stream");
    }

}
```

#### 访问

启动消息消费者，运行单元测试，消息消费者控制台打印结果如下：

![](入门测试.png)

RabbitMQ 界面如下：

![](RabbitMQ-入门测试.png)

![](RabbitMQ-入门测试-1.png)

## 8. 自定义消息通道

### 创建消息通道

参考源码`Source.java`和`Sink.java`创建自定义消息通道。

自定义消息发送者通道`MySource.java`

```java
package com.springcloud.demo.channel;

import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

public interface MySource {

    String MY_OUTPUT = "my_output";

    @Output(MY_OUTPUT)
    MessageChannel myOutput();
}
```

自定义消息接收者通道`MySink.java`

```java
package com.springcloud.demo.channel;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

public interface MySink {

    String MY_INPUT = "my_input";

    @Input(MY_INPUT)
    SubscribableChannel myInput();
}
```

### 配置文件

消息生产者

```yaml
server:
  port: 8001 # 端口

spring:
  application:
    name: stream-producer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息发送通道
        # 与 org.springframework.cloud.stream.messaging.Source 中的 @Output("output") 注解的 value 相同
        output:
          destination: stream.message # 绑定的交换机名称
        my_output:
          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

消息消费者

```java
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称
        my_input:
          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

### 代码重构

消息生产者

`MessageProducer.java`

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.custom.MySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(MySource.class)
// @EnableBinding(Source.class)
public class MessageProducer {

//    @Autowired
//    private Source source;
    @Autowired
    private MySource source;

    /**
     * 发送消息
     *
     * @param message
     */
    public void send(String message) {
        source.myOutput().send(MessageBuilder.withPayload(message).build());
    }

}
```

消息消费者

`MessageConsumer.java`

```java
package com.springcloud.demo.consumer;

import com.springcloud.demo.custom.MySink;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.stereotype.Component;

/**
 * 消息消费者
 */
@Component
@EnableBinding(MySink.class)
//@EnableBinding(Sink.class)
public class MessageConsumer {

    /**
     * 接收消息
     *
     * @param message
     */
    //@StreamListener(Sink.INPUT)
    @StreamListener(MySink.MY_INPUT)
    public void receive(String message) {
        System.out.println("message = " + message);
    }

}
```

### 测试

#### 单元测试

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.StreamProducerApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(classes = {StreamProducerApplication.class})
public class MessageProducerTest {


    @Autowired
    private MessageProducer messageProducer;

    @Test
    public void testSend() {
        messageProducer.send("hello spring cloud stream");
    }

}
```

#### 访问

启动消息消费者，运行单元测试，消息消费者控制台打印结果如下：

```java
message = hello spring cloud stream
```

RabbitMQ 界面如下：

![](RabbitMQ-自定义通道-测试.png)

## 9. 配置优化



Spring Cloud 微服务开发之所以简单，除了官方做了许多彻底的封装之外还有一个优点就是**约定大于配置**。开发人员仅需规定应用中不符约定的部分，在没有规定配置的地方采用默认配置，以力求最简配置为核心思想。

> 简单理解就是：Spring 遵循了推荐默认配置的思想，当存在特殊需求时候，自定义配置即可否则无需配置。

在 Spring Cloud Stream 中，`@Output("output")` 和 `@Input("input")` 注解的 `value` 默认即为绑定的交换机名称。所以自定义消息通道的案例我们就可以重构为以下方式。

### 创建消息通道

参考`Source.java`和	`Sinkling.java`创建自定义消息通道。

自定义消息通道`MyDefaultSource.java`

```java
package com.springcloud.demo.channel;

import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

public interface MyDefaultSource {

    String MY_OUTPUT = "default.message";

    @Output(MY_OUTPUT)
    MessageChannel myOutput();
}
```

自定义消息接收通道`MyDefaultSink.java`

```java
package com.springcloud.demo.channel;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

public interface MyDeaultSink {

    String DEFAULT_INPUT = "default.message";

    @Input(DEFAULT_INPUT)
    SubscribableChannel defaultInput();
}
```

### 配置文件

消息生产者

```yaml
server:
  port: 8001 # 端口

spring:
  application:
    name: stream-producer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
#  cloud:
#    stream:
#      bindings:
#        # 消息发送通道
#        # 与 org.springframework.cloud.stream.messaging.Source 中的 @Output("output") 注解的 value 相同
#        output:
#          destination: stream.message # 绑定的交换机名称
#        my_output:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

消息消费者

```yaml
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
#  cloud:
#    stream:
#      bindings:
#        # 消息接收通道
#        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
#        input:
#          destination: stream.message # 绑定的交换机名称
#        my_input:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

### 代码重构

消费生产者

`MessageDefaultProducer.java`

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.custom.MyDefaultSource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(MyDefaultSource.class)
public class MessageDefaultProducer {

    @Autowired
    private MyDefaultSource source;

    public void send(String message){
        source.defaultOutput().send(MessageBuilder.withPayload(message).build());

    }
}
```

消息消费者

`MessageDefaultConsumer.java`

```java
package com.springcloud.demo.consumer;

import com.springcloud.demo.custom.MyDeaultSink;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(MyDeaultSink.class)
public class MessageDefaultConsumer {

    @StreamListener(MyDeaultSink.DEFAULT_INPUT)
    public void receive(String message) {
        System.out.println("message = " + message);
    }
}
```

### 测试

#### 单元测试

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.StreamProducerApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(classes = {StreamProducerApplication.class})
public class MessageProducerTest {


    @Autowired
    private MessageProducer messageProducer;

    @Autowired
    private MessageDefaultProducer messageDefaultProducer;

    @Test
    public void testSend() {
        messageProducer.send("hello spring cloud stream");
    }

    @Test
    public void testDefaultSend(){
        messageDefaultProducer.send("约定大于配置！");
    }
}
```

#### 访问

启动消息消费者，运行单元测试，消息消费者控制台打印结果如下：

![](优化配置-测试.png)

RabbitMQ 界面如下：

![](配置优化-测试-1.png)

## 10. 短信邮件发送案例

一个消息驱动微服务应用可以既是消息生产者又是消息消费者。接下来模拟一个短信邮件发送的消息处理过程：

- 原始消息发送至 `source.message` 交换机；
- 消息驱动微服务应用通过 `source.message` 交换机接收原始消息，经过处理分别发送至 `sms.message` 和 `email.message` 交换机；
- 消息驱动微服务应用通过 `sms.message` 和 `email.message` 交换机接收处理后的消息并发送短信和邮件。


### 创建消息通道

发送原始消息，接收处理后的消息并发送短信和邮件的消息驱动微服务应用。

```java
package com.springcloud.demo.channel;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.SubscribableChannel;

public interface MyProcessor {

    String SOURCE_MESSAGE = "source.message";
    String SMS_MESSAGE = "sms.message";
    String EMAIL_MESSAGE = "email.message";

    @Output(SOURCE_MESSAGE)
    MessageChannel sourceOutput();

    @Input(SMS_MESSAGE)
    SubscribableChannel smsInput();

    @Input(EMAIL_MESSAGE)
    SubscribableChannel emailInput();
}
```

接收原始消息，经过处理分别发送短信和邮箱的消息驱动微服务应用。

```java
package com.springcloud.demo.channel;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.SubscribableChannel;

public interface MyProcessor {

    String SOURCE_MESSAGE = "source.message";
    String SMS_MESSAGE = "sms.message";
    String EMAIL_MESSAGE = "email.message";

    @Input(SOURCE_MESSAGE)
    MessageChannel sourceOutput();

    @Output(SMS_MESSAGE)
    SubscribableChannel smsOutput();

    @Output(EMAIL_MESSAGE)
    SubscribableChannel emailOutput();

}
```

### 配置文件

约定大于配置，配置文件只修改端口和应用名称即可，其他配置一致。

```yaml
server:
  port: 8001 # 端口

spring:
  application:
    name: stream-producer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
#  cloud:
#    stream:
#      bindings:
#        # 消息发送通道
#        # 与 org.springframework.cloud.stream.messaging.Source 中的 @Output("output") 注解的 value 相同
#        output:
#          destination: stream.message # 绑定的交换机名称
#        my_output:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```



```yaml
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
#  cloud:
#    stream:
#      bindings:
#        # 消息接收通道
#        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
#        input:
#          destination: stream.message # 绑定的交换机名称
#        my_input:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

### 消息驱动服务A

> 发送原始消息，通过 `sms.message` 和 `email.message` 交换机接收处理后的消息并发送短信和邮件

#### 发送消息

发送原始消息 `10000|10000@email.com` 至 `source.message` 交换机。

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.channel.MyProcessor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@EnableBinding(MyProcessor.class)
public class SourceMessageProducer {

    @Autowired
    private MyProcessor myProcessor;

    /**
     * 发送原始消息
     *
     * @param sourceMessage
     */
    public void send(String sourceMessage) {
        log.info("原始消息发送成功，原始消息为：{}", sourceMessage);
        myProcessor.sourceOutput().send(MessageBuilder.withPayload(sourceMessage).build());
    }
}
```

#### 接收消息

接收处理后的消息并发送短信和邮件。

```java
package com.springcloud.demo.consumer;

import com.springcloud.demo.channel.MyProcessor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@EnableBinding(MyProcessor.class)
public class SmsAndEmailMessageConsumer {

    /**
     * 接收消息 电话号码
     *
     * @param phoneNum
     */
    @StreamListener(MyProcessor.SMS_MESSAGE)
    public void receiveSms(String phoneNum) {
        log.info("电话号码为：{}，调用短信发送服务，发送短信...", phoneNum);
    }

    /**
     * 接收消息 邮箱地址
     *
     * @param emailAddress
     */
    @StreamListener(MyProcessor.EMAIL_MESSAGE)
    public void receiveEmail(String emailAddress) {
        log.info("邮箱地址为：{}，调用邮件发送服务，发送邮件...", emailAddress);
    }
}
```

### 消息驱动服务B

#### 接收消息

接收原始消息 `10000|10000@email.com` 处理后并发送至 `sms.message` 和 `email.message` 交换机。

```java
package com.springcloud.demo.consumer;

import com.springcloud.demo.channel.MyProcessor;
import com.springcloud.demo.producer.SmsAndEmailMessageProducer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.stereotype.Component;

/**
 * 消息消费者
 */
@Slf4j
@Component
@EnableBinding(MyProcessor.class)
public class SourceMessageConsumer {

    @Autowired
    private SmsAndEmailMessageProducer smsAndEmailMessageProducer;

    /**
     * 接收原始消息，处理后并发送
     *
     * @param sourceMessage
     */
    @StreamListener(MyProcessor.SOURCE_MESSAGE)
    public void receive(String sourceMessage) {
        log.info("原始消息接收成功，原始消息为：{}", sourceMessage);
        // 发送消息 电话号码
        smsAndEmailMessageProducer.sendSms(sourceMessage.split("\\|")[0]);
        // 发送消息 邮箱地址
        smsAndEmailMessageProducer.sendEmail(sourceMessage.split("\\|")[1]);
    }
}
```

#### 发送消息

发送电话号码 `10000` 和邮箱地址 `10000@email.com` 至 `sms.message` 和 `email.message` 交换机。

```java
package com.springcloud.demo.consumer;

import com.springcloud.demo.channel.MyProcessor;
import com.springcloud.demo.producer.SmsAndEmailMessageProducer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.stereotype.Component;

/**
 * 消息消费者
 */
@Slf4j
@Component
@EnableBinding(MyProcessor.class)
public class SourceMessageConsumer {

    @Autowired
    private SmsAndEmailMessageProducer smsAndEmailMessageProducer;

    /**
     * 接收原始消息，处理后并发送
     *
     * @param sourceMessage
     */
    @StreamListener(MyProcessor.SOURCE_MESSAGE)
    public void receive(String sourceMessage) {
        log.info("原始消息接收成功，原始消息为：{}", sourceMessage);
        // 发送消息 电话号码
        smsAndEmailMessageProducer.sendSms(sourceMessage.split("\\|")[0]);
        // 发送消息 邮箱地址
        smsAndEmailMessageProducer.sendEmail(sourceMessage.split("\\|")[1]);
    }
}
```

### 测试

#### 单元测试

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.StreamProducerApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(classes = {StreamProducerApplication.class})
public class MessageProducerTest {


    @Autowired
    private MessageProducer messageProducer;

    @Autowired
    private MessageDefaultProducer messageDefaultProducer;

    @Autowired
    private SourceMessageProducer sourceMessageProducer;



    @Test
    public void testSend() {
        messageProducer.send("hello spring cloud stream");
    }

    @Test
    public void testDefaultSend(){
        messageDefaultProducer.send("约定大于配置！");
    }

    @Test
    public void testSendSource() {
        sourceMessageProducer.send("10000|10000@email.com");
    }
}
```

#### 访问

消息驱动微服务 A 控制台打印结果如下：

```shell
2022-06-20 23:16:29.117  INFO 23724 --- [faRjgsuIojLRw-1] c.s.d.c.SmsAndEmailMessageConsumer       : 邮箱地址为：10000@email.com，调用邮件发送服务，发送邮件...
2022-06-20 23:16:29.121  INFO 23724 --- [fifeIzGnC4BUw-1] c.s.d.c.SmsAndEmailMessageConsumer       : 电话号码为：10000，调用短信发送服务，发送短信...
```

消息驱动微服务 B 控制台打印结果如下：

```java
2022-06-20 23:15:28.700  INFO 5048 --- [yW_i6Ldwwydsg-1] c.s.d.p.SmsAndEmailMessageProducer       : 邮箱地址消息发送成功，消息为：10000@email.com
2022-06-20 23:16:29.096  INFO 5048 --- [yW_i6Ldwwydsg-1] c.s.demo.consumer.SourceMessageConsumer  : 原始消息接收成功，原始消息为：10000|10000@email.com
2022-06-20 23:16:29.096  INFO 5048 --- [yW_i6Ldwwydsg-1] c.s.d.p.SmsAndEmailMessageProducer       : 电话号码消息发送成功，消息为：10000
2022-06-20 23:16:29.097  INFO 5048 --- [yW_i6Ldwwydsg-1] c.s.d.p.SmsAndEmailMessageProducer       : 邮箱地址消息发送成功，消息为：10000@email.com
```

RabbitMQ 界面如下：

![](短信邮件实例.png)

## 11. 消息分组

如果有多个消息消费者，那么消息生产者发送的消息会被多个消费者都接收到，这种情况在某些实际场景下是有很大问题的，比如在如下场景中，订单系统做集群部署，都会从 RabbitMQ 中获取订单信息，如果一个订单消息同时被两个服务消费，系统肯定会出现问题。为了避免这种情况，Stream 提供了消息分组来解决该问题。

![](消费分组.png)

在 Stream 中处于同一个 `group` 中的多个消费者是竞争关系，能够保证消息只会被其中一个应用消费。不同的组是可以消费的，同一个组会发生竞争关系，只有其中一个可以消费。通过 `spring.cloud.stream.bindings.<bindingName>.group` 属性指定组名。

![](消费分组-1.png)

### 问题演示

在 `stream-demo` 项目下创建 `stream-consumer1 子项目。

项目代码使用入门案例中消息消费者的代码。

#### 消息生产者

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.channel.MySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
//s@EnableBinding(MySource.class)
 @EnableBinding(Source.class)
public class MessageProducer {

    @Autowired
    private Source source;
//    @Autowired
//    private MySource source;

    /**
     * 发送消息
     *
     * @param message
     */
    public void send(String message) {
        source.output().send(MessageBuilder.withPayload(message).build());
        //source.outPut().send(MessageBuilder.withPayload(message).build());
    }

}
```

配置文件

```yaml
server:
  port: 8001 # 端口

spring:
  application:
    name: stream-producer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息发送通道
        # 与 org.springframework.cloud.stream.messaging.Source 中的 @Output("output") 注解的 value 相同
        output:
          destination: stream.message # 绑定的交换机名称
#        my_output:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

#### 消息消费者

```java
package com.springcloud.demo.consumer;

import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.stereotype.Component;

/**
 * 消息消费者
 */
@Component
@EnableBinding(Sink.class)
public class MessageConsumer {

    /**
     * 接收消息
     *
     * @param message
     */
    @StreamListener(Sink.INPUT)
    public void receive(String message) {
        System.out.println("message = " + message);
    }

}
```

配置文件

`stream-consumer`配置文件

```yaml
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称
#        my_input:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

```

`stream-consumer1`配置文件

```yaml
server:
  port: 8003 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称


# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

```



### 单元测试

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.StreamProducerApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(classes = {StreamProducerApplication.class})
public class MessageProducerTest {


    @Autowired
    private MessageProducer messageProducer;



    @Test
    public void testSend() {
        messageProducer.send("hello spring cloud stream");
    }
}

```

运行单元测试发送消息，两个消息消费者控制台打印结果如下：

`stream-consumer` 的控制台：

![](消息分组-测试-1.png)

`stream-consumer1`的控制台：

![](消息分组-测试-2.png)

通过结果可以看到消息被两个消费者同时消费了，原因是因为它们属于不同的分组，默认情况下分组名称是随机生成的，通过 RabbitMQ 也可以得知：

![](消息分组-测试-3.png)

### 配置分组

stream-consumer 的分组配置为：`group-stream-consumer`。

```yaml
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称
          group: group-stream-consumer
#        my_input:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/

```

stream-consumer 的分组配置为：`group-stream-consumer`。

```yaml
server:
  port: 8003 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称
          group: group-stream-consumer



# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```



### 测试 

运行单元测试发送消息，此时多个消息消费者只有其中一个可以消费。RabbitMQ 结果如下：

![](消息分组-测试-4.png)

## 12. 消息分区

> [Preface (spring.io)](https://docs.spring.io/spring-cloud-stream/docs/3.1.6/reference/html/spring-cloud-stream.html#spring-cloud-stream-overview-partitioning)

有一些场景需要满足, 同一个特征的数据被同一个实例消费, 比如同一个id的传感器监测数据必须被同一个实例统计计算分析, 否则可能无法获取全部的数据。又比如部分异步任务，首次请求启动task，二次请求取消task，此场景就必须保证两次请求至同一实例。

![](消息分区.png)

### 问题演示

消息未分区的效果，单元测试代码如下：

```java
/**
     * 消息分区测试
     */
    @Test
    public void testSendPartition() {
        for (int i = 1; i <= 10; i++) {
            messageProducer.send("hello spring cloud stream");
        }
    }
```

### 测试

运行单元测试发送消息，两个消息消费者控制台打印结果如下：

`stream-consumer`控台打印如下：

```shell
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
```

`stream-consumer1`控制台打印如下：

```shell
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
message = hello spring cloud stream
```

假设这 10 条消息都来自同一个用户，正确的方式应该都由一个消费者消费所有消息，否则系统肯定会出现问题。为了避免这种情况，Stream 提供了消息分区来解决该问题。

### 配置分区

消息生产者配置**分区键的表达式规则**和**消息分区的数量**。

```yaml
server:
  port: 8001 # 端口

spring:
  application:
    name: stream-producer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      bindings:
        # 消息发送通道
        # 与 org.springframework.cloud.stream.messaging.Source 中的 @Output("output") 注解的 value 相同
        output:
          destination: stream.message # 绑定的交换机名称
          producer:
            partition-key-expression: payload # 配置分区键的表达式规则
            partition-count: 2 # 配置消息分区的数量
#        my_output:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

通过 `partition-key-expression` 参数指定分区键的表达式规则，用于区分每个消息被发送至对应分区的输出 `channel`。

该表达式作用于传递给 `MessageChannel` 的 `send` 方法的参数，该参数实现 `org.springframework.messaging.Message` 接口的 `GenericMessage` 类。

#### 源码解读

![](消息分区-spring-messaging-Message.png)

源码 MessageChannel.java

```java
/*
 * Copyright 2002-2016 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.messaging;

/**
 * Defines methods for sending messages.
 *
 * @author Mark Fisher
 * @since 4.0
 */
@FunctionalInterface
public interface MessageChannel {

	/**
	 * Constant for sending a message without a prescribed timeout.
	 */
	long INDEFINITE_TIMEOUT = -1;


	/**
	 * Send a {@link Message} to this channel. If the message is sent successfully,
	 * the method returns {@code true}. If the message cannot be sent due to a
	 * non-fatal reason, the method returns {@code false}. The method may also
	 * throw a RuntimeException in case of non-recoverable errors.
	 * <p>This method may block indefinitely, depending on the implementation.
	 * To provide a maximum wait time, use {@link #send(Message, long)}.
	 * @param message the message to send
	 * @return whether or not the message was sent
	 */
	default boolean send(Message<?> message) {
		return send(message, INDEFINITE_TIMEOUT);
	}

	/**
	 * Send a message, blocking until either the message is accepted or the
	 * specified timeout period elapses.
	 * @param message the message to send
	 * @param timeout the timeout in milliseconds or {@link #INDEFINITE_TIMEOUT}
	 * @return {@code true} if the message is sent, {@code false} if not
	 * including a timeout of an interrupt of the send
	 */
	boolean send(Message<?> message, long timeout);

}
```

源码 GenericMessage.java

```java
/*
 * Copyright 2002-2018 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.messaging.support;

import java.io.Serializable;
import java.util.Map;

import org.springframework.lang.Nullable;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.util.Assert;
import org.springframework.util.ObjectUtils;

/**
 * An implementation of {@link Message} with a generic payload.
 * Once created, a GenericMessage is immutable.
 *
 * @author Mark Fisher
 * @since 4.0
 * @param <T> the payload type
 * @see MessageBuilder
 */
public class GenericMessage<T> implements Message<T>, Serializable {

	private static final long serialVersionUID = 4268801052358035098L;


	private final T payload;

	private final MessageHeaders headers;


	/**
	 * Create a new message with the given payload.
	 * @param payload the message payload (never {@code null})
	 */
	public GenericMessage(T payload) {
		this(payload, new MessageHeaders(null));
	}

	/**
	 * Create a new message with the given payload and headers.
	 * The content of the given header map is copied.
	 * @param payload the message payload (never {@code null})
	 * @param headers message headers to use for initialization
	 */
	public GenericMessage(T payload, Map<String, Object> headers) {
		this(payload, new MessageHeaders(headers));
	}

	/**
	 * A constructor with the {@link MessageHeaders} instance to use.
	 * <p><strong>Note:</strong> the given {@code MessageHeaders} instance is used
	 * directly in the new message, i.e. it is not copied.
	 * @param payload the message payload (never {@code null})
	 * @param headers message headers
	 */
	public GenericMessage(T payload, MessageHeaders headers) {
		Assert.notNull(payload, "Payload must not be null");
		Assert.notNull(headers, "MessageHeaders must not be null");
		this.payload = payload;
		this.headers = headers;
	}


	@Override
	public T getPayload() {
		return this.payload;
	}

	@Override
	public MessageHeaders getHeaders() {
		return this.headers;
	}


	@Override
	public boolean equals(@Nullable Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof GenericMessage)) {
			return false;
		}
		GenericMessage<?> otherMsg = (GenericMessage<?>) other;
		// Using nullSafeEquals for proper array equals comparisons
		return (ObjectUtils.nullSafeEquals(this.payload, otherMsg.payload) && this.headers.equals(otherMsg.headers));
	}

	@Override
	public int hashCode() {
		// Using nullSafeHashCode for proper array hashCode handling
		return (ObjectUtils.nullSafeHashCode(this.payload) * 23 + this.headers.hashCode());
	}

	@Override
	public String toString() {
		StringBuilder sb = new StringBuilder(getClass().getSimpleName());
		sb.append(" [payload=");
		if (this.payload instanceof byte[]) {
			sb.append("byte[").append(((byte[]) this.payload).length).append("]");
		}
		else {
			sb.append(this.payload);
		}
		sb.append(", headers=").append(this.headers).append("]");
		return sb.toString();
	}

}
```

如果 `partition-key-expression` 的值是 `payload`，将会使用所有放在 `GenericMessage` 中的数据作为分区数据。`payload` 是消息的实体类型，可以为自定义类型比如 `User`，`Role` 等等。

如果 `partition-key-expression` 的值是 `headers["xxx"]`，将由 `MessageBuilder` 类的 `setHeader()` 方法完成赋值，比如：

```java
package com.springcloud.demo.producer;

import com.springcloud.demo.channel.MySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
//s@EnableBinding(MySource.class)
 @EnableBinding(Source.class)
public class MessageProducer {

    @Autowired
    private Source source;
//    @Autowired
//    private MySource source;

    /**
     * 发送消息
     *
     * @param message
     */
    public void send(String message) {
        source.output().send(MessageBuilder.withPayload(message).setHeader("xxx", 0).build());
        //source.output().send(MessageBuilder.withPayload(message).build());
        //source.outPut().send(MessageBuilder.withPayload(message).build());
    }

}
```

消息消费者配置**消费者总数**和**当前消费者的索引**并**开启分区支持**。

`stream-consumer` 的 `application.yml`

```yaml
server:
  port: 8002 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      instance-count: 2 # 消费者总数
      instance-index: 0 # 当前消费者的索引
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称
          group: group-stream-consumer # 消息分组
          consumer:
            partitioned: true # 开启分区支持
#        my_input:
#          destination: my.message # 绑定的交换机名称

# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```

`stream-consumer1`的`application.yml`

```yaml
![消息分区-测试-1](Spring Cloud Stream消息驱动/消息分区-测试-1.png)server:
  port: 8003 # 端口

spring:
  application:
    name: stream-consumer # 应用名称
  rabbitmq:
    host: 192.168.56.56  # 服务器 IP
    port: 5672            # 服务器端口
    username: guest       # 用户名
    password: guest       # 密码
    virtual-host: /       # 虚拟主机地址
  cloud:
    stream:
      instance-count: 2 # 消费者总数
      instance-index: 1 # 当前消费者的索引
      bindings:
        # 消息接收通道
        # 与 org.springframework.cloud.stream.messaging.Sink 中的 @Input("input") 注解的 value 相同
        input:
          destination: stream.message # 绑定的交换机名称
          group: group-stream-consumer # 消息分组
          consumer:
            partitioned: true # 开启分区支持



# 配置 Eureka Server 注册中心
eureka:
  instance:
    prefer-ip-address: true       # 是否使用 ip 地址注册
    instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  client:
    service-url:                  # 设置服务注册中心地址
      defaultZone: http://root:123456@localhost:8762/eureka/,http://root:123456@localhost:8763/eureka/
```



### 测试

运行单元测试发送消息，此时多个消息消费者只有其中一个可以消费所有消息。RabbitMQ 结果如下：

![](消息分区-测试-1.png)

![](消息分区-测试-2.png)



至此Stream 消息驱动所有的知识点就学习结束了。

---

参考资料：

[Spring Cloud Stream - duanxz - 博客园 (cnblogs.com)](https://www.cnblogs.com/duanxz/p/11929321.html)

[Spring Cloud Stream - 天宇轩-王 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dalianpai/p/12300738.html)

[Spring Cloud 之 Stream. - JMCui - 博客园 (cnblogs.com)](https://www.cnblogs.com/jmcui/p/11279388.html)

[Spring Cloud 系列之 Stream 消息驱动 - 哈喽沃德先生 (mrhelloworld.com)](https://mrhelloworld.com/stream/)

[SpringCloud进阶：消息驱动之Spring Cloud Stream 消息分区 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/107142676)

