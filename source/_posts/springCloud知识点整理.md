---
title: springCloud知识点整理
date: 2022-08-23 09:39:25
tags: 
- 整理
- 分布式
categories: 微服务
---

## 1. 为什么会出现SpringCloud?

单体架构的优缺点

优点：

- 部署简单：只有一个包
- 技术单一：同一个架构来说，基本上是一个一种技术，类似springboot + java
- 用人成本低：因为只需要一种技术

缺点：

- 代码结构混乱：业务复杂，导致代码量很大，管理会越来越困难。
- 经常需要解决冲突，开发效率低：开发人员同时维护同一套代码，很难避免代码冲突。开发过程中会伴随解决冲突，严重影响开发效率。
- 排查解决问题成本高：线上发现了bug，可能bu g很简单，但是由于只有一套代码，需要编译打包上线，成本很高。
- 监控很难：很多功能点，杂在一个系统中。

由于单体结构的应用随着系统复杂度的增高，会暴露出各种各样的问题。近些年来，微服务架构逐渐取代了单体架构，且这种趋势将会越来越流行。



## 2. **什么是Spring Cloud**？

SpringCloud是基于SpringBoot的一整套实现微服务的**框架**。它提供了微服务开发所需的服务注册与发现,服务调用,负载均衡,API网关

配置管理,熔断器

**springCloud的优缺点**

优点：

- 产出于Spring大家族，Spring在企业级开发框架中无人能敌，来头很大，可以保证后续的更新、完善 。
- 组件丰富，功能齐全。Spring Cloud 为微服务架构提供了非常完整的支持。例 如、配置管理、服务发现、断路器、微服务网关等； 
- Spring Cloud 社区活跃度很高，教程很丰富，遇到问题很容易找到解决方案
- 服务拆分粒度更细，耦合度比较低，有利于资源重复利用，有利于提高开发效率
- 减轻团队的成本，可以并行开发，不用关注其他人怎么开发，先关注自己的开发 
- 微服务可以是跨平台的，可以用任何一种语言开发 

缺点：

- 微服务过多，治理成本高，不利于维护系统 
- 分布式系统开发的成本高（容错，分布式事务等）对团队挑战大 

## 3. springboot 和springCloud 的区别和联系

- springCloud 是基于Springboot 构建的，他里面的每个子项目就是一个springboot项目。SpringBoot可以离开SpringCloud独立使用开发项目， 但是SpringCloud离不 开SpringBoot ，属于依赖的关系 
- SpringBoot专注于快速方便的开发单个个体微服务。 SpringCloud是关注全局的微服务协调整理治理框架，为微服务架构提供了非常完整的支持。例如、配置管理、服务发现、断路器、微服务网关等

## 4. **Spring Cloud Netflix**核心组件？

Netflix OSS 开源组件集成，包括Eureka、Hystrix、Ribbon、Feign、Zuul等 核心组件。

- Eureka：服务治理组件，包括服务端的注册中心和客户端的服务发现机制； 

- Ribbon：负载均衡的服务调用组件，具有多种负载均衡调用策略； 

- Feign：基于Ribbon和Hystrix的声明式服务调用组件； 

- Zuul：API网关组件，对请求提供路由及过滤功能。

- Hystrix：服务容错组件，实现了断路器模式，为依赖服务的出错和延迟提供了 容错能力； 

- Bus: 

# 5. 微服务架构组成

## 5.1 服务注册与发现

### 1. 服务注册和发现是什么

- 服务注册 ，就是将提供某个服务的模块信息 (通常是这个服务的ip和端口)注册到1个公共的组件上去。
- 服务发现 ，就是新**注册的这个服务模块能够及时的被其他调用者发现。**

### 2. 什么是注册中心

注册中心可以说是微服务架构中的”通讯录“，它记录了服务和服务地址的映射关系。在分布式架构中，服务会注册到这里，当服务需要调用其它服务时，就到这里找到服务的地址，进行调用。

### 3 注册中心的实现方式

- mysql: 将服务和服务地址的映射关系insert到数据库中，调用服务前去select。维护 一个心跳，定期去感应下节点状态是否正常，比如ping一下端口，如果不正常，修改此时节点状态
- zookeeper：在这个父节点上创建临时子节点，用来存储服务和服务地址的映射关系，对应生成注册表。调用服务前去节点里查询节点数据，zookeeper有监听通知机制，我们对某个节点进行监听，当这个节点被删除时，监听方会感知到重新拉取服务列表。
- Eureka

### 4. Spring Cloud 如何实**现服务注册和发现？**

由于所有服务都在 Eureka 服务器上注册并 通过调用 Eureka 服务器完成查找，因此无需处理服务地点的任何更改和处理。

[CAP原则以及eureka和zookeeper的对比](https://pyr9.github.io/CAP%E5%8E%9F%E5%88%99%E4%BB%A5%E5%8F%8Aeureka%E5%92%8Czookeeper%E7%9A%84%E5%AF%B9%E6%AF%94/)

### 5 eureka注册中心是每次调用都从注册中心拉吗？

Eureka 客户端**不会每次调用都去注册中心拉取服务列表**，它是**本地缓存 + 定时增量更新**的机制。

#### 1. 工作原理

**1. 启动时全量拉取一次** 客户端启动后会向 Eureka Server 拉取所有服务实例列表，缓存在本地内存中。

**2. 之后每 30 秒增量更新一次（默认）** 客户端每隔一段时间（`eureka.client.registry-fetch-interval-seconds`，默认 30s）向 Server 发请求，只拉取**变化的那部分实例信息**，然后合并到本地缓存。

**3. 本地有故障时才重新全量拉取** 如果长时间连不上 Server，客户端会继续用**本地缓存的服务列表**来做负载均衡和调用，直到连接恢复。

#### 2. 本地缓存的作用

**1. 减少注册中心压力**：不用每次调用都去查 Server。

**2. 提高可用性**：Server 短暂不可用时，客户端仍能根据缓存继续调用已有服务实例。

**3. 加速读取**：直接从本地内存拿服务列表，比远程调用快得多。

#### 3. 配置相关参数（Spring Cloud 为例）

```yml
eureka:
  client:
    registry-fetch-interval-seconds: 30   # 多久拉一次增量（默认30s）
    fetch-registry: true                  # 是否要从Server拉取注册表（默认true）
```

如果你把这个设为很大（比如 300s），那么本地缓存更新会变慢，服务上下线感知也会延迟。

## 5.2 服务调用



## 5.3 负载均衡



## 5.4 API网关

### Spring Cloud Gateway 和zuul

相同点：二者都是可以用来实现API网关

不同点：

- 开发团队不同。
  - zuul是netfix开发的
  - spring Cloud Gateway是Spring Cloud官方推出的第二代网关框架，用来取代Zuul网关。
- zuul性能相对gateway较差。
  - Zuul1.* 是一个极具阻塞IO的API GateWay，每次IO操作都是从工作线程中选择一个执行，请求线程被阻塞到工作线程完成。所以zuul的性能较差。zuu2理念更先进，基于netty非阻塞，但是springCloud还没有整合。
  - Spring Cloud Gateway 是基于 WebFlux 框架实现的，而 WebFlux 框架底层则使用了高性能的 Reactor 模式通信框架 Netty。高并发和非阻塞通信就非常有优势
  - Spring Cloud Gateway还支持webSocket，并且与spring有很好的集成

> WebSocket 协议,大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于[服务器推送技术](https://en.wikipedia.org/wiki/Push_technology)的一种。

 多方面比较，gateWay是很理想的网关选择。



## 5.5 配置管理

### 1 springCloud config 是什么？

- Spring Cloud Config 是由 Spring Cloud 团队开发的项目，它可以为微服务架构中各个微服务提供集中化的外部配置支持。
- Spring Cloud Config 可以将各个微服务的配置文件集中存储在一个外部的存储仓库或系统（例如 Git 、SVN 等）中，对配置的统一管理，以支持各个微服务的运行。
- Spring Cloud Config 包含以下两个部分：
  - Config Server：也被称为分布式配置中心，它是一个独立运行的微服务应用，用来连接配置仓库并为客户端提供获取配置信息、加密信息和解密信息的访问接口。
  - Config Client：指的是微服务架构中的各个微服务，它们通过 Config Server 对配置进行管理，并从 Config Sever 中获取和加载配置信息。

## 5.6 熔断器

### 1. 什么是服务熔断？

- 服务熔断就是对服务的调用执行熔断，对于后续的请求，不再调用目标服务，而是直接返回一个合理的信息，比如说请求量太大，返回一个”此时服务过载”，从而可以快速释放资源
- 保护系统

### 2 为什么会有这种熔断机制的出现?

在微服务相互调用的时候可能会出现服务调用发生异常或调用超时,多此请求造成服务积压导致服务无法使用后接着级联导致调用此服务的消费者也出现异常或调用超时,最后导致整个系统的全部崩溃.这就是雪崩效应.

> [级联](https://so.csdn.net/so/search?q=级联&spm=1001.2101.3001.7020)(失败)雪崩：一个服务失败，导致整条链路的服务都失败的情形。

### 3. 熔断器的功能

- 异常处理。需要可以根据异常的类型，去处理异常
- 日志记录。记录失败的数目，才可以开启熔断
- 测试失败的操作。周期性的检查来测试服务是否健康。
- 手动复位。管理员可以强行关闭断路器，重置故障的计数器
- 并发。断路器可以支持大并发量
- 加速断路：检测出问题的速度要快，及时响应
- 重试失败请求：在服务可用时，可以安排重试

### 4 熔断和降级的区别

相同点：

- 目的一致。通过技术手段来保护系统
- 表现类似：让用户体验到某些服务暂时不可用
- 粒度相同：都是基于服务的

不同点

- 触发条件不同
  - 服务熔断：服务熔断由链路上某个服务故障引起的
  - 服务降级，整体负荷来考虑。比如：原本有一个系统可以承载100个请求，但是后面由于请求数降低，相应的服务数量也需要降低
- 管理目标层级不同
  - 服务熔断：服务熔断是一个框架层次的处理，
  - 服务降级：服务降级是业务层次的处理。降级一般是从外围服务开始

### 5. 熔断的意义？

- 让系统更加稳定
- 减少性能损耗
  - 响应很简单，所以产生的性能很小。
  - 用户知道服务有问题了，就不再调用，避免一直重试，也可以减少性能损耗。
- 及时响应。不需要其他计算，直接就可以返回一个简单的响应。
- 阀值可定制，比如请求到达10000，就开启熔断器

### 6. Spring Cloud 如何实**现服务服务熔断**

- Feign是Netflix开发的一款服务容错和保护组件
- 当出现服务异常或调用超时时，Hystrix 定义了一个回退方法，与公开服 务相同的返回类型。如果暴露服务中出现异常，则回退方法将返回一些值。

```java
  @RequestMapping("/cities")
  @HystrixCommand(fallbackMethod="defaultCities")
  public String listCity(){
    return cityClient.listCity();
  }

  public String defaultCities(){
    return "Micro weather city eureka is down!";
  }
```

## 5.7 消息总线

### 1. Spring Cloud Bus

- Spring Cloud Bus 又被称为消息总线，它能够通过轻量级的消息代理（例如 RabbitMQ、Kafka 等）来构建一个公用的**消息主题**，将微服务架构中的各个服务都连接上来起来，从而实现广播状态更改、事件推送等功能，还可以实现微服务之间的通信功能。
- 目前 Spring Cloud Bus 支持两种消息代理：RabbitMQ 和 Kafka。



## Spring Cloud Bus 的基本原理

- Spring Cloud Bus 会使用一个轻量级的消息代理来构建一个公共的消息主题 Topic（默认为“springCloudBus”），**这个 Topic 中的消息会被所有服务实例监听和消费**。
- 当其中的一个服务刷新数据时，Spring Cloud Bus 会把信息保存到 Topic 中，这样监听这个 Topic 的服务就收到消息并自动消费。



## Spring Cloud Bus 动态刷新配置的原理

- 当 Git 仓库中的配置发生了改变，我们只需要向某一个服务Config 服务端，也可以是 Config 客户端发送一个 POST 请求，Spring Cloud Bus 就可以通过消息代理通知其他服务重新拉取最新配置，以实现配置的动态刷新。
- 利用 Spring Cloud Bus 的特殊机制可以实现很多功能，其中配合 Spring Cloud Config 实现配置的动态刷新就是最典型的应用场景之一。

## Spring Cloud Bus 整合Spring Cloud Config实现配置的动态刷新

1. 当 Git 仓库中的配置发生改变后，运维人员向 Config 服务端发送一个 POST 请求，请求路径为“/actuator/refresh”。
2. Config 服务端接收到请求后，会将该请求转发给服务总线 Spring Cloud Bus。
3. Spring Cloud Bus 接到消息后，会通知给所有 Config 客户端。
4. Config 客户端接收到通知，请求 Config 服务端拉取最新配置。
5. 所有 Config 客户端都获取到最新的配置。





### Ribbon 和 Feign 的区别和联系

- 相同点：二者都是Netflix开发的，可以用来实现服务间的调用以及负载均衡。Feign 是在 Ribbon的基础上进行了一次改进，是一个使用起来更加方便的 HTTP 客户端。

- 调用方式不同

  - ribbon是一个基于 HTTP 和 TCP **客户端**的负载均衡的工具。简单来说，rubbon = 负载均衡+ restTemplate，需要自己构建请求，步骤相当繁琐。

    ```java
     //面向微服务编程，即通过微服务的名称来获取调用地址
      private static final String REST_URL_PREFIX = "http://micro-weather-city-eureka";
    
      @RequestMapping("/cities")
      public String listCity(){
        return restTemplate.getForObject(REST_URL_PREFIX + "/cities", String.class);
      }
    ```

    

  - 采用接口的方式， **`只需要创建一个接口，然后在上面添加注解@FeignClient即可`** ，将需要调用的其他服务的方法定义成抽象方法即可， 不需要自己构建http请求。

    ```java
    @FeignClient("micro-weather-city-eureka")
    public interface CityClient {
      @RequestMapping("/cities")
      String listCity();
    }
    ```

- 启动类上使用的注解不同

  Ribbon用的是@RibbonClient，Feign用的是@EnableFeignClients。

 由于Ribbon写起来还需要自己拼请求，所以采用了更简单的Feign







