---
title: CAS单点登录前后端分离
date: 2018-07-31 21:56:20
tags: spring
---

## 关于单点登录(SSO)

假设用户X需同时登录站点A和站点B，这两个站点之间其实是有关联性的，但是如果用户认证数据不通用，那将需要注册或登录两次。

单点登录系统就是为了解决这种场景的问题，建立一种用户认证中心，只要经过这个中心注册或登录了某一站点服务的用户，总是能够认证登录这个中心所授权的其他所有服务。

<!--more-->

## 关于CAS(Central Authentication Service)

CAS是一种单点登录开源框架，遵循apache2.0协议，代码托管在github.com/apereo/cas上。

CAS作为一种单点登录框架，后端可配置不同的用户数据库，支持自定义验证或加密逻辑，并提供不同的协议用于与业务server(cas-client)间的通信。

CAS的源码是由java写的，因此对于java的web项目天生友好。当然只要实现CAS相关协议的client，无论是哪种语言实现的，都能集成到CAS认证框架中。

## 关于CAS前后端分离

CAS5.x版默认的注册登录相关UI是以html形式内嵌在CAS-web项目中的，其中结合thymeleaf模板库来渲染html。

我们也可以通过前后端分离的方式来自定义我们的认证UI，我这里采取了spring+vuejs架构进行了测试一下，基本能跑通认证流程，大概过程用一张时序图来说明：
![图片描述][1]

## 前后端分离所涉及到的restAPI

因为CAS由原来的结构改为了springboot+vuejs, 原有访问CAS页面的url要修改成访问vuejs所构建的前端页面，而这些url则由这些自定义的前端页面代理去访问调用。这些api包括:

+ 通过（用户名|密码|当前service名)获取TGT票据, 如：

```shell

curl -v -X POST "https://svr.sso.com/cas/v1/tickets" -d "username=testuser&password=testpwd&service=http://xxx/xx"

```

响应的TGT信息存储在Location中。

+ 通过TGT获取ST，如：

```shell
// 参数service需要进行urlencode
curl -X POST -v "https://svr.sso.com/cas/v1/tickets/TGT-4-xxxxxxx" -d "service=http%3a%2f%2fxxx%3a%2fxx"
```

这时直接能从响应结果中得到ST字符串。

+ 通过ST访问service，正常下可完成登录:

```
http://xxx/xx?ticket=ST-3-BGJBL1wWjRw-prwoSYiyQaYzUNkAppledeMacBook-Pro
```

+ 注销登录(清除TGT)，单点登出，如：

```shell
curl -X DELETE -v "https://svr.sso.com/cas/v1/tickets/TGT-94-xxxxx"
```

正常情况下会返回已被清理的TGT字段。

  [1]: //img.mukewang.com/5b6028770001f6f214361182.png
