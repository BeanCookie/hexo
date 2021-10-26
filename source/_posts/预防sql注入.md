---
title: 预防sql注入
date: 2020-08-19 10:10:13
categories:
- 网络安全
tags:
- sql注入
---

#### 什么是SQL注入

> 通过在请求参数中传入特殊字符，从而绕过正常的SQL逻辑获取非法数据

#### SQL注入实例

1. 这是一段正常的登录验证sql，正常情况下platform_uk会传入手机号，只有password输入正确之后才能获取用户数据

```sql
SELECT
	* 
FROM
	passport 
WHERE
	platform_uk = '*******9355' AND password = '*******yykEY7zG';
```

但是如果一些恶意攻击者传入一些恶意的platform_uk，在没有sql防注入的情况下会出现严重的信息泄露问题，比如传入的platform_uk为**18120139355' OR 1 = 1 -- '**，这种情况下就可以直接绕过密码验证获取登录权限

```sql
SELECT
	* 
FROM
	passport 
WHERE
	platform_uk = '18120139355' OR 1 = 1 -- ' and password = '*******yykEY7zG';
```

其实用户权限相关的sql注入现在已经很难实现，各种语言的框架都会使用防注入的策略避免以上问题。

#### 框架是如何避免sql注入

试着找了Laravel之前的一个漏洞

[laravel-query-builder中的严重SQL注入漏洞](https://shieldfy.io/blog/serious-sql-injection-vulnerability-in-laravel-query-builder/)

> [https://cr-api.zhgcloud.com/cp-web/demands?page=1&count_per_page=10?sort=created_at-%3E%22%27))%23or1=1](https://cr-api.zhgcloud.com/cp-web/demands?page=1&count_per_page=10?sort=created_at->"'))%23or1=1)

修复之后会直接报一个错误

> ```
> [2020-08-19 11:18:57.934483] local.ERROR: A non well formed numeric value encountered {"exception":"[object] (ErrorException(code: 0): A non well formed numeric value encountered at /opt/sites/saas-rental/vendor/laravel/framework/src/Illuminate/Database/Query/Builder.php:1552)"} 
> ```

#### 编码中如何避免被注入sql

1. 避免编写原生SQL，所有参数都需要通过编译传入，Laravel中严禁使用whereRaw方法

   > Eloquent 内部使用的是 PDO 参数绑定，所以你的请求是安全的。虽然如此，在一些允许你使用原生 SQL 语句的地方，还是要特别小心，例如 whereRaw 或者 selectRaw 。如下：
   >
   > User::whereRaw("name = '$input_name'")->first();
   > 以上这段代码里是存在安全漏洞的，请尽量避免使用原生查询语句。如果你有必须使用原生语句的情况，语句里又包含用户提交内容的话，可以利用其提供的，类似于 PDO 参数绑定进行传参，以避免 SQL 注入的风险：
   >
   > User::whereRaw("name = ?", [$input_name])->first();
   >

2. 优先使用ORM访问数据库

