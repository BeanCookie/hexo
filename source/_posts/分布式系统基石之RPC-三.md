---
title: 分布式系统基石之RPC(三)
date: 2020-05-12 10:56:10
categories:
- 分布式
tags:
- RPC
---

## 分布式系统基石之RPC(三)-Dubbo实现细节

### 如何自定义协议

> 大多数的自定义协议底层依然是基于TCP协议，为了提高效率TCP会采用流式传输，相较于面向消息的传输TCP会出现粘包问题。

![分布式系统基石之RPC-06](https://beancookie.github.io/images/分布式系统基石之RPC-06.png)

![分布式系统基石之RPC-07](https://beancookie.github.io/images/分布式系统基石之RPC-07.png)

#### 通常有三种解决方案

- 在报文尾部增加换行符

  > LineBasedFrameDecoder

- 讲消息分为消息头、消息体，可以在消息头中声明消息的长度

  > DelimiterBasedFrameDecoder

- 规定好报文长度，不足的空位补齐

  > FixedLengthFrameDecoder

#### 实现拆包之后还需要定义具体协议

- JSON
- Protocol Buffer
- Thrift
- Dubbo

### Zookeeper概览

> ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services
>
> ZooKeeper是用于维护配置信息，命名，提供分布式同步和提供组服务的集中式服务

![分布式系统基石之RPC-08](https://beancookie.github.io/images/分布式系统基石之RPC-08.png)

#### 临时节点

> 当创建节点的程序停止时，Zookeeper会自动删除该节点

#### 监听机制

> Watch的整体流程如下图所示，客户端先向ZooKeeper服务端成功注册想要监听的节点状态，同时客户端本地会存储该监听器相关的信息在WatchManager中，当ZooKeeper服务端监听的数据状态发生变化时，ZooKeeper就会主动通知发送相应事件信息给相关会话客户端，客户端就会在本地响应式的回调相关Watcher的Handler。

![分布式系统基石之RPC-09](https://beancookie.github.io/images/分布式系统基石之RPC-09.png)

### 服务发现

> 从理论角度理解服务发现，首先讨论[观察者模式](http://c.biancheng.net/view/1390.html)

#### Zookeeper的特性

> Zookeeper虽然提供了一个树形的数据结构，但是Zookeeper并不能像Redis支撑大量请求，这是因为Zookeeper主要作用是协调分布式服务。所以实现RPC时不会在每次RPC请求之前都去访问一次Zookeeper，而是在本地内存中缓存一份服务提供者的数据，当服务提供者节点发送变化时才去更新本地的缓存列表。



##### Links

https://www.apache.org/index.html#projects-list

https://willemjiang.github.io/opensource/2018/10/21/asf-introduction.html

https://juejin.im/entry/5c09de8af265da616c656980