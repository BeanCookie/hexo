---
title: JWT原理
date: 2020-07-08 14:52:47
categories:
- 认证
tags:
- JWT
---

## JWT原理

服务端认证完用户之后会生成一个`JSON`格式的`Token`发送给用户

**`Token`格式如下**

```json
{
  "姓名": "张三",
  "角色": "管理员",
  "到期时间": "2020年8月1日0点0分"
}
```

在此之后每次客户端请求服务端时都会带上这个`Token`，服务端只依靠此`Token`进行身份验证，为了防止信息配篡改`Token`中会加上信息签名。

#### JWT格式

实际上的`JWT`格式如下

`eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ0ZXN0IiwiaWF0IjoxNTk0MTkzODUwLCJleHAiOjE1OTQxOTU2NTB9.-HuaBI2JmXimeP55Vb8rUuB8yBPe9BMObdSvycTQ85nLwBCbFs2VlhSw0DqkO42i4dgn8yfjh0FynRSHQNKSWA`

它只是一个很长的字符串，其中用`.`分隔成三部分

- **Header（头部）**
- **Payload（负载）**
- **Signature（签名）**

可以通过访问[http://calebb.net/](http://calebb.net/)解析这段`JWT`字符串

```
{
	alg: "HS512"
}.
{
	sub: "test",
	iat: 1594193850,
	exp: 1594195650
}.
[signature]
```

#### Header 

header部分是一个 JSON 对象，描述 JWT 的元数据，`alg`属性表示签名的算法（algorithm），默认是 HMAC SHA256（写成 HS256），`typ`属性表示这个令牌（token）的类型（type），JWT 令牌统一写为`JWT`

####  Payload

ayload 部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段，供选用

1. iss (issuer)：签发人
2. exp (expiration time)：过期时间
3. sub (subject)：主题
4. aud (audience)：受众
5. nbf (Not Before)：生效时间
6. iat (Issued At)：签发时间
7. jti (JWT ID)：编号

#### Signature

Signature 部分是对前两部分的签名，防止数据篡改。

伪代码如下

```javascript
HMACSHA256
(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload) + "." +
  secret
)
```

首先，需要指定一个密钥（secret）。这个密钥只有服务器才知道，不能泄露给用户。然后，使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。算出签名以后，把 **Header**、**Payload**、**Signature** 三个部分拼成一个字符串，每个部分之间用"点"（`.`）分隔，就可以返回给用户

### `JWT`的特点

- `JWT` 默认是不加密，但也是可以加密的。生成原始` Token` 以后，可以用密钥再加密一次
- `JWT` 不加密的情况下，不能将秘密数据写入` JWT`
- `JWT `的最大缺点是，由于服务器不保存 `session `状态，因此无法在使用过程中废止某个 `Token`，或者更改 `Token` 的权限。也就是说，一旦 `JWT` 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑
- `JWT `本身包含了认证信息，一旦泄露，任何人都可以获得该令牌的所有权限。为了减少盗用，`JWT `的有效期应该设置得比较短。对于一些比较重要的权限，使用时应该再次对用户进行认证
- 为了减少盗用，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输

