---
title: MySQL事务
date: 2024-04-29 10:03
tags:
- MySQL
categories:
- 学习
---

由于系统需要做动态路由，所以每个角色的权限列表都很长，导致后端生成的 JWT token 也很长。通过 debug 后发现 Spring Security 默认的类 JwtAccessTokenConverter 的 enhance 方法会把权限列表的值进行 encode，如下图所示：
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/8822eb6c9b1fc09fb683ea1e90fcf38b.png)
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/da0e2a40b87159321a5705bc9bd263ce.png)
encode 后会形成一个很长的 token，考虑到系统并没有直接从 token 中解析获取 token，而是获取 Spring Security 的上下文对象再从数据库获取获取这个用户的权限：
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/db061e21c595fde4f2a8f2e4734eb1a6.png)
所以决定去除 token 中的权限列表。

解决的方法也很简单，上述的 encode 方法中会调用 tokenConverter 的 convertAccessToken 方法，如下图所示：
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/c57f32a15ed288098d56cd901759bc52.png)

查看代码发现这个类调用的是默认的 DefaultAccessTokenConverter，并且提供了 set 方法，所以只需要在这个类加载完后，也就是生命周期的初始化后调用 set 方法把 tokenConverter 改成我们自己的类就可以了

![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/086b0618e4c2388a7f55c560b40c91f6.png)

所以这个时候需要实现一个我们自己的 tokenConverter，由于我们只是要改动 DefaultAccessTokenConverter 的 convertAccessToken 方法，改变其返回的值就可以了，所以直接继承 DefaultAccessTokenConverter 类。
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/30b50fc9e928c0bfc15cea0249af2a58.png)

观察 DefaultAccessTokenConverter 的 convertAccessToken 可以看出，response 里面 put 了 key 为 UserAuthenticationConverter.AUTHORITIES 的权限数据。
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/681a6978cae092d3b464769465d833d6.png)

所以我们只需要在我们自己的类中 remove 掉这个 key 就可以了。
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/f4f405e9f5404eafc0b4dbb7df983ea3.png)

最后在我们自己的 JwtAccessTokenConverter 中实现 bean 的初始化生命周期函数类，实现它的初始化后方法并设置我们上面实现的 tokenConverter 就可以了。要注意的是需要加上 @DependsOn(value = "platformJWTAccessTokenConverter") 防止该类初始化的时候 Spring 容器还未注入我们自己实现的 tokenConverter。
![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/1a515156e825f6026c05a8ce7f77bd8d.png)
