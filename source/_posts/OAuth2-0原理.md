---
title: OAuth2.0原理
date: 2020-07-14 10:29:57
categories:
- 认证
tags:
- OAuth2.0
---

## OAuth 2.0原理

#### 定义

[OAuth](http://en.wikipedia.org/wiki/OAuth)是一个关于授权（authorization）的开放网络标准，在全世界得到广泛应用，目前的版本是2.0版。

#### 应用场景

假设项目中想使用微信授权登录。此时OAuth就会在"客户端"与"服务提供商"之间，设置了一个授权层（authorization layer）。"客户端"不能直接登录"服务提供商"，只能登录授权层，以此将用户与客户端区分开来。"客户端"登录授权层所用的令牌（token），与用户的密码不同。用户可以在登录的时候，指定授权层令牌的权限范围和有效期。

"客户端"登录授权层以后，"服务提供商"根据令牌的权限范围和有效期，向"客户端"开放用户储存的资料。

#### 名词概念

- **Third-party application**：第三方应用程序，本文中又称"客户端"（client），放在智鹤就是指挥官项目
- **HTTP service**：HTTP服务提供商，本文中简称"服务提供商"，即上一节例子中的微信
- **Resource Owner**：资源所有者，本文中又称"用户"（user）
- **User Agent**：用户代理，本文中就是指浏览器
- **Authorization server**：认证服务器，即服务提供商专门用来处理认证的服务器
- **Resource server**：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器

#### 运行流程

```
+--------+                               +---------------+
|        |--(A)- Authorization Request ->|   Resource    |
|        |                               |     Owner     |
|        |<-(B)-- Authorization Grant ---|               |
|        |                               +---------------+
|        |
|        |                               +---------------+
|        |--(C)-- Authorization Grant -->| Authorization |
| Client |                               |     Server    |
|        |<-(D)----- Access Token -------|               |
|        |                               +---------------+
|        |
|        |                               +---------------+
|        |--(E)----- Access Token ------>|    Resource   |
|        |                               |     Server    |
|        |<-(F)--- Protected Resource ---|               |
+--------+                               +---------------+
```

- （A）用户打开客户端以后，客户端要求用户给予授权
- （B）**用户同意给予客户端授权**。（核心步骤）
- （C）客户端使用上一步获得的授权，向认证服务器申请令牌
- （D）认证服务器对客户端进行认证以后，确认无误，同意发放令牌
- （E）客户端使用令牌，向资源服务器申请获取资源
- （F）资源服务器确认令牌无误，同意向客户端开放资源

#### 微信授权登录

![https://beancookie.github.io/images/OAuth2.0原理-01.png](https://beancookie.github.io/images/OAuth2.0原理-01.png)

#### 参考资料

- http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html
- https://developers.weixin.qq.com/doc/oplatform/Website_App/WeChat_Login/Wechat_Login.html