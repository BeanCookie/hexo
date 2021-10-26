---
title: 分布式系统基石之RPC(二)
date: 2020-04-26 19:58:16
categories:
- 分布式
tags:
- RPC
---

## 分布式系统基石之RPC(二)

### RPC调用流程

![https://beancookie.github.io/images/分布式系统基石之RPC-04.png](https://beancookie.github.io/images/分布式系统基石之RPC-04.png)

### 协议

> ​	网络上只能传输二进制的数据，RPC把请求发送到网络之前，首先需要把请求参数转换成二进制格式，然后写入本地Socket再由网卡发送到网络设备。但是TCP协议会将二进制拆分成多个包，所以接收方无法根据二进制数据解析成一个完整的请求。可以理解为生活当中的断句，如果没有统一的协议就相当于<u>句读之不知，惑之不解</u>

- HTTP

  > HTTP1.1协议的初衷就是一个无状态的，用于在浏览器和服务器之间进行单项请求的协议，但是由于无法进行双向通信，以及协议设计比较冗余在RPC环境下弊端愈发明显

- SPDY

  > 2012年由Google提出的一个优化方案，虽然2015年被移除但是为HTTP2奠定了基础

  1. 多路复用
  2. 请求头压缩
  3. 服务端推送
  4. 强制使用TLS加密

- HTTP2
  1. HTTP1.x的解析是基于文本，HTTP2采用二进制的格式，扩展了协议的应用场景
  2. 多路复用，与SPDY一样将一个TCP连接分为若干个流（Stream），每个流中可以传输若干消息（Message）
  3. 使用HPACK算法压缩请求头
  4. 服务端推送
  5. TLS加密变成了可选项

### 系列化

- JSON

  > JSON虽然是JavaScript的一个子集，但却是一种语言独立的文本协议

  ```json
  {
       "firstName": "John",
       "lastName": "Smith",
       "sex": "male",
       "age": 25,
       "address": 
       {
           "streetAddress": "21 2nd Street",
           "city": "New York",
           "state": "NY",
           "postalCode": "10021"
       },
       "phoneNumber": 
       [
           {
             "type": "home",
             "number": "212 555-1234"
           },
           {
             "type": "fax",
             "number": "646 555-4567"
           }
       ]
   }
  ```

  

- Protocol Buffer

  > 需要使用IDL文件定义消息格式，再用protoc工具转化成各种语言对应的类或结构体用于编程

  ```protobuf
  syntax = "proto3";
  
  message Person {
    string firstName = 1;
    string lastName = 2;
    string sex = 3;
    double age = 4;
  
    message Address {
      string streetAddress = 1;
      string city = 2;
      string state = 3;
      string postalCode = 4;
    }
  
    Address address = 5;
  
    message PhoneNumber {
      string type = 1;
      string number = 2;
    }
  
    repeated PhoneNumber phoneNumber = 6;
  }
  
  ```

### 调用方式

![https://beancookie.github.io/images/分布式系统基石之RPC-05.png](https://beancookie.github.io/images/分布式系统基石之RPC-05.png)

- [x] https://github.com/BeanCookie/grpc-python
- [x] https://github.com/BeanCookie/grpc-java

- 同步阻塞式

  ```java
  private final WaterCompanyGrpc.WaterCompanyBlockingStub futureStub;
  
  public WaterBlockingClient(Channel channel) {
      futureStub = WaterCompanyGrpc.newBlockingStub(channel);
  }
  
  public void butWater(float price) {
      WaterRequest request = WaterRequest.newBuilder().setPrice(price).build();
      WaterReply waterReply;
      try {
          waterReply = futureStub.buyWater(request);
      } catch (StatusRuntimeException e) {
          logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
      }
  }
  
  ```

- 异步非阻塞

  ```java
  private final WaterCompanyGrpc.WaterCompanyFutureStub futureStub;
  
  public WaterFutureClient(Channel channel) {
      futureStub = WaterCompanyGrpc.newFutureStub(channel);
  }
  
  public void butWater(float price) {
      WaterRequest request = WaterRequest.newBuilder().setPrice(price).build();
      Future<WaterReply> waterReplyFuture;
      try {
          waterReplyFuture = futureStub.buyWater(request);
      } catch (StatusRuntimeException e) {
          logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
      } catch (InterruptedException | ExecutionException e) {
          e.printStackTrace();
      }
  }
  ```

- Streaming

  ```java
  private final WaterCompanyGrpc.WaterCompanyStub futureStub;
  
  final CountDownLatch latch = new CountDownLatch(1);
  
  public WaterStreamClient(Channel channel) {
      futureStub = WaterCompanyGrpc.newStub(channel);
  }
  
  public void butWater(float price) {
      WaterRequest request = WaterRequest.newBuilder().setPrice(price).build();
      try {
          futureStub.buyStreamWater(request, new StreamObserver<WaterReply>() {
              @Override
              public void onNext(WaterReply waterReply) {
                  logger.info("WaterCompany: " + waterReply.getMessage());
              }
  
              @Override
              public void onError(Throwable throwable) {
                  logger.log(Level.WARNING, throwable.getMessage());
              }
  
              @Override
              public void onCompleted() {
                  logger.info("WaterCompany: completed");
                  latch.countDown();
              }
          });
          latch.await();
      } catch (StatusRuntimeException e) {
          logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
  
  }
  ```

  