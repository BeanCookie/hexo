---
title: 分布式系统基石之RPC(一)
date: 2020-04-18 17:53:59
categories:
- 分布式
tags:
- RPC
---

## 分布式系统基石之RPC(一)

### 什么是RPC

> RPC的全称是Remote Procedure Call(远程过程调用)，也就是说两台服务器A和B，一个部署在A服务器上的应用，想要调用部署在B服务器上某个应用提供的函数，由于不在一个内存空间不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。

### 举个栗子

```java
package com.zeaho.ms.dubbo.provider.diy;

@RestController
@RequestMapping("/diy/provider")
public class AppleProvider {
    @GetMapping("/apple")
    public AppleDTO getApple() {
        return new AppleDTO(1, "V1", "DIYApple", LocalDateTime.now());
    }
}

```

```java
package com.zeaho.ms.dubbo.consumer.diy;

@RestController
@RequestMapping("/diy/v2")
@Configuration
@Slf4j
public class V2AppleConsumer {

    @Value("${diy.remote.host}")
    private String remoteHost;

    @Value("${diy.remote.port}")
    private Integer remotePort;

    @Value("${diy.services.rpcMethod}")
    private String rpcMethod;

    @GetMapping("/apple")
    public AppleDTO apple() {
        String url = String.format("http://%s:%d%s", remoteHost, remotePort, rpcMethod);
        return new RestTemplate().getForObject(url, AppleDTO.class);
    }
}
```

#### 优点

- 调用简单天生支持跨语言
- HTTP协议大家都理解，使用JSON序列化数据也很方便

#### 缺点

- 通过URL来描述方法，用接口文档来描述请求和相应数据在跨部门沟通时容易产生问题
- 单点的HTTP接口难以保障7*24提供文档的服务
- 传输效率与其他方案比较并非最优

### 基本的概念

- Spring Boot

  > Java生态中MVC时代的集大成者，有两个最主要的特质：1.约定大于配置 2.内置Web服务器

- Spring Cloud

  > 借用农夫山泉的广告词，我们不生成微服务，我们只做微服务的搬运工

- Dubbo

  > 可以理解为Spring Cloud中的一个子集，本质上只是一个RPC框架，阿里重启之后呢做了很多适配工作，现在Dubbo中可以集成很多微服务的解决方案
### 真正能够用于实际的RPC方案还需要一个很重要的特性

> #### 像调用本地方法一样调用远程接口

### 服务发布

1. RESTful API（Spring Cloud）

   > ```xml
   > # 公用的请求响应对象
   > <dependency>
   > <groupId>com.zeaho.ms</groupId>
   > <artifactId>api</artifactId>
   > <version>${api.version}</version>
   > </dependency>
   > ```
   >
   > ```java
   > # 服务提供方就是一个简单的RESTful接口
   > package com.zeaho.ms.cloud.provider.controller;
   > 
   > @RestController
   > @RequestMapping("/provider")
   > public class AppleProvider {
   >  @GetMapping("/apple")
   >  public AppleDTO getApple() {
   >      return new AppleDTO(1, "V1", "DIYApple", LocalDateTime.now());
   >  }
   > 
   > }
   > ```
   >
   > ```java
   > # 在Spring Cloud中服务消费方使用OpenFeign来调用接口
   > package com.zeaho.ms.cloud.consumer.feign;
   > 
   > @FeignClient(name = "cloud-provider", fallback = AppleServiceFallback.class)
   > public interface AppleService {
   >     /**
   >      *
   >      * @return AppleDTO
   >      * @author luzhong
   >      * @since 2020-04-02 15:56
   >      */
   >     @GetMapping("/provider/apple")
   >     AppleDTO getApple();
   > }
   > ```
   >
   > ```java
   > package com.zeaho.ms.cloud.consumer.controller;
   > 
   > @RestController
   > @RequestMapping("/cloud/v1")
   > public class AppleController {
   >     @Autowired
   >     private AppleService appleService;
   > 
   >     @GetMapping("/apple")
   >     public AppleDTO apple() {
   >         return appleService.getApple();
   >     }
   > }
   > ```

2. interface（Dubbo）

   > ```xml
   > # 公用的请求响应对象和接口
   > <dependency>
   > <groupId>com.zeaho.ms</groupId>
   > <artifactId>api</artifactId>
   > <version>${api.version}</version>
   > </dependency>
   > ```
   >
   > ```java
   > # 公用的消息接口
   > package com.zeaho.ms.cloud.api.v1;
   > 
   > import com.zeaho.ms.cloud.api.dto.AppleDTO;
   > 
   > /**
   >  * @author lzzz
   >  * @since 2020-02-27 11:23
   >  */
   > public interface AppleService {
   >     /**
   >      * 获取一个苹果
   >      * @return
   >      */
   >     AppleDTO getApple();
   > }
   > ```
   >
   > ```java
   > # 服务提供方是一个接口的实现类
   > package com.zeaho.ms.dubbo.provider.service.impl;
   > 
   > @Service(version = "1.0", loadbalance = "roundrobin")
   > @Component
   > public class AppleServiceImpl implements AppleService {
   >     @Autowired
   >     Environment environment;
   > 
   >     @Override
   >     public AppleDTO getApple() {
   >        return new AppleDTO(1, "V2", "DubboAppple:" + environment.getProperty("local.server.port"), LocalDateTime.now());
   >     }
   > }
   > ```
   >
   > ```java
   > # Dubbo和OpenFeign类似也是通过调用interface来调用接口
   > package com.zeaho.ms.dubbo.consumer.dubbo;
   > 
   > @RestController
   > @RequestMapping("/dubbo/v2")
   > public class V2AppleController {
   > 
   >     @Reference(version = "1.0")
   >     private AppleService appleService;
   > 
   >     @GetMapping("/apple")
   >     private AppleDTO apple() {
   >         return appleService.getApple();
   >     }
   > }
   > ```
   
   
   
3. IDL文件（interface description language），通过一种中立的方式来描述接口，使得在不同的平台上运行的对象和不同语言编写的程序可以相互通信交流。比如你用Java语言实现提供的一个服务也能被PHP语言调用。

   > ```idl
   > service AppleService {
   >   rpc getApple () returns (Apple) {}
   > }
   > 
   > message Apple {
   >   string message = 1;
   > }  
   > ```
   >
   > 假如服务提供者使用的是Java语言，那么利用protoc插件即可自动生成Server端的Java代码。
   >
   > 
   >
   > 假如服务消费者使用的是PHP语言，那么利用protoc插件即可自动生成Client端的PHP代码。

### 服务注册与发现

#### 注册中心有什么用？

> 从点外卖说起，如果时间倒流5年在所有外卖平台还未出现之前，在宿舍想点份沙县小吃的外卖，这时候我们拿出了
>
> ![https://beancookie.github.io/images/分布式系统基石之RPC-01.png](https://beancookie.github.io/images/分布式系统基石之RPC-01.png)
> 这里主要信息有两个外卖电话和各种小吃，这里的电话可以理解为服务IP+端口号，各种小吃就是各个方法。当我们拨打电话然后点某份小吃的过程就是一次RPC。及时沙县再好吃也有吃腻的一天，比如今天我想吃兰州拉面但是外卖卡找不到了，或者哪天餐厅的电话或者小吃种类变了，这时没法正常点外卖了，同样服务的IP地址、端口号或者方法列表有改动RPC调用也会失败。
>
> 现在我们怎么点外卖，![https://beancookie.github.io/images/分布式系统基石之RPC-02.png](https://beancookie.github.io/images/分布式系统基石之RPC-02.png)
> 现在我们不需要手机每个店铺的点餐卡了，只要打开一个外卖APP就可以看到外卖店列表和店里的各种餐点。这时外卖平台会帮我们维护餐厅和菜品列表，我们只需要下单就可以了。回到我们的代码，注册中心起到了同样的作用，服务提供方会在启动时将自己所能提供的服务发布到注册中心，当服务信息发生变动的时候会通知配置中心进行更改，当注册中心检测到服务不稳定时也可以将服务下线，服务消费方调用服务时从注册中心获取提供者的信息，注册中心的高可用由其自身提供。

### 注册中心原理

![https://beancookie.github.io/images/分布式系统基石之RPC-03.png](https://beancookie.github.io/images/分布式系统基石之RPC-03.png)

- RPC Server提供服务，在启动时向Registry注册自身服务，并向Registry定期发送心跳汇报存活状态。
- RPC Client调用服务，在启动时向Registry订阅服务，把Registry返回的服务节点列表缓存在本地内存中，并与RPC Sever建立连接。
- 当RPC Server节点发生变更时，Registry会同步变更，RPC Client感知后会刷新本地内存中缓存的服务节点列表。
- RPC Client从本地缓存的服务节点列表中，基于负载均衡算法选择一台RPC Sever发起调用。

### 常见的几种注册中心

> 选择注册中心之前不需要考虑的两个问题
>
> 1. **高可用性**
>
>    > 注册中心作为服务提供者和服务消费者之间沟通的纽带，它的高可用性十分重要
>    >
>    > 多实例
>    >
>    > 多机房
>
> 2. **数据一致性**
>
>    > 首先会涉及到著名的CAP理论（Consistency一致性、Availability可用性、Partition Tolerance分区容错性）这里有个前提一个分布式系统不可能同时满足CAP三种条件，只能选择其中两条。
>    >
>    > 一致性：数据在多个副本之间能否保证一致性，执行更新操作后，应该保证系统的数据仍然处于一致的状态。
>    >
>    > 可用性：有限时间内返回结果
>    >
>    > 分区容错性：遇到任何网络分区故障的时候，任然能够对外提供满足一致性和可用性的服务。
>    >
>    > 虽然CAP理论有三条，但是如果要避免出现分区容错性问题就需要将所有数据都放在同一节点上，这样就会导致系统失去扩展性。所以通常我们只在CP和AP中做选择

- Eureka

  > 采用应用内注册与发现的方式，不幸的是Netflix已经将其闭源了

- Zookeeper

  > 一个极其成熟稳定的分布式协调服务，其中有个很重要的Paxos协议
  >
  > https://www.cnblogs.com/hzmark/p/The_Part-Time_Parliament.html

- Nacos

  > 阿里推出的服务发现和配置管理解决方案

### 服务治理

>  引入RPC之后服务调用由本地调用变成远程调用，服务消费者A需要通过注册中心去查询服务提供者B的地址，然后发起调用，这个看似简单的过程就可能会遇到下面几种情况

- 注册中心宕机
- 服务提供者B有节点宕机
- 服务消费者A和注册中心之间的网络不通
- 服务提供者B和注册中心之间的网络不通
- 服务消费者A和服务提供者B之间的网络不通
- 服务提供者B有些节点性能变慢
- 服务提供者B短时间内出现问题

#### 负载均衡

- 负载类型
  1. 服务端反向代理
  2. 客户端负载均衡

1. 随机算法
2. 轮询算法
3. 最少活跃调用算法（LRU）
4. 一致性Hash算法

#### 服务路由

- 灰度发布

  > 服务提供者做了功能变更，但希望先只让部分人群使用，然后根据这部分人群的使用反馈，再来决定是否做全量发布

#### 服务容错

- 限流

  > 根据服务器性能和服务特性每个接口的限流阈值实际上是不同的

  - 流量控制类型

  1. QPS
  2. 并发线程数流量控制

  - 控制的效果

  1. 直接拒绝

  2. Warm Up：预热/冷启动方式。当系统长期处于低水位的情况下，当流量突然增加时，直接把系统拉升到高水位可能瞬间把系统压垮。通过"冷启动"，让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮

  3. 匀速排队：严格控制请求通过的间隔时间，也即是让请求以均匀的速度通过，对应的是漏桶算法

- 降级

  > 通过停止系统中的某些功能，来保证系统整体的可用性
  
- 熔断
#### 节点管理

> 因为注册中心实现了高可用，那么问题大概率会出现在服务提供者自身和网络问题

- 注册中心主动摘除机制

  > 这种机制要求服务提供者定时的主动向注册中心汇报心跳，注册中心根据服务提供者节点最近一次汇报心跳的时间与上一次汇报心跳时间做比较，如果超出一定时间，就认为服务提供者出现问题，继而把节点从服务列表中摘除，并把最近的可用服务节点列表推送给服务消费者

- 服务消费者摘除机制

  > 虽然注册中心主动摘除机制可以解决服务提供者节点异常的问题，但如果是因为注册中心与服务提供者之间的网络出现异常，最坏的情况是注册中心会把服务节点全部摘除，导致服务消费者没有可用的服务节点调用，但其实这时候服务提供者本身是正常的。所以，将存活探测机制用在服务消费者这一端更合理，如果服务消费者调用服务提供者节点失败，就将这个节点从内存中保存的可用服务提供者节点列表中移除