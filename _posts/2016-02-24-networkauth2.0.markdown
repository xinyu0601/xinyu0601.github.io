---
title: Auth2.0学习小结
layout: post
date: 2016-02-24 22:44
image: "/assets/images/markdown.jpg"
tag:
- 网络
- Auth2.0
category: blog
author: johndoe
description: Markdown summary with different options
---

## Auth2.0是什么
***Auth2.0是一种授权框架，仅用于授权代理。***OAuth并没有支持HTTP以外的协议；OAuth并不是一个认证协议；OAuth并没有定义授权处理机制；OAuth并没有定义token 格式；OAuth2.0并没有定义加密方式；OAuth2.0并不是单个协议；***OAuth2.0仅是授权框架，仅用于授权代理。***
#### 解决的问题领域和场景
- 开放系统间授权
	- 社交联合登录
	- 开放API平台
- 现代微服务安全
	- 单页浏览器App
	- 无限原生App
	- 服务端WebApp
	- 微服务和API间调用 
- 企业内部应用认证授权

#### OAuth术语

Third-party application：第三方应用程序，本文中又称"客户端"（client），即上一节例子中的"云冲印"。

HTTP service：HTTP服务提供商，本文中简称"服务提供商"，即上一节例子中的Google。

Resource Owner：资源所有者，本文中又称"用户"（user）。

User Agent：用户代理，本文中就是指浏览器。

Authorization server：认证服务器，即服务提供商专门用来处理认证的服务器。

Resource server：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

![](/auth2.0.JPG)

## OAth令牌类型
**授权码**

仅用于授权码授权类型，用于交
换获取访问令牌和刷新令牌

**刷新令牌**

用于去授权服务器获取一个新的
访问令牌

**访问令牌**

用于代表一个用户或服务直接
去访问受保护的资源

**Bearer Token**

不管谁拿到Token都可以
访问资源，像现钞

**Proof of Possession Token**

可以校验client是否对Token
有明确的拥有权

## 总结

* OAuth 本质： 如何获取token, 如何使用token
* OAuth提供一个宽泛的协议框架，具体安全场景需要定制
* OAuth是一种在系统之间的代理授权协议
* OAuth使用代理协议的方式解决密码共享反模式问题



## OAuth2.0 经典模式

客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。

1. 授权码模式（authorization code）
1. 简化模式（implicit）
1. 密码模式（resource owner password credentials）
1. 客户端模式（client credentials）

#### 授权码模式
授权码模式（authorization code）是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器，与"服务提供商"的认证服务器进行互动。
![](/bg2014051204.png)
它的步骤如下：
1. 用户访问客户端，后者将前者导向认证服务器。
1. 用户选择是否给予客户端授权。
1. 假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。
1. 客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。
1. 认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。


#### 简化模式
简化模式（implicit grant type）不通过第三方应用程序的服务器，直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此得名。所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。

![](/bg2014051205.png)

#### 密码模式
密码模式（Resource Owner Password Credentials Grant）中，用户向客户端提供自己的用户名和密码。客户端使用这些信息，向"服务商提供商"索要授权。

在这种模式中，用户必须把自己的密码给客户端，但是客户端不得储存密码。这通常用在用户对客户端高度信任的情况下，比如客户端是操作系统的一部分，或者由一个著名公司出品。而认证服务器只有在其他授权模式无法执行的情况下，才能考虑使用这种模式。

![](http://www.ruanyifeng.com/blogimg/asset/2014/bg2014051206.png)


#### 客户端模式

客户端模式（Client Credentials Grant）指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。严格地说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。

![](http://www.ruanyifeng.com/blogimg/asset/2014/bg2014051207.png)


它的步骤如下：

（A）客户端向认证服务器进行身份认证，并要求一个访问令牌。

（B）认证服务器确认无误后，向客户端提供访问令牌。