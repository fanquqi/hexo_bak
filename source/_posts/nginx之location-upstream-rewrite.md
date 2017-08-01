---
title: 'nginx之location,upstream,rewrite'
date: 2017-07-08 14:19:45
tags: nginx
categories: 基础运维
---

> nginx配置第三篇，[nginx安装配置](https://fanquqi.github.io/2017/07/06/nginx%E4%B9%8B%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/)中讲了很多基础配置，这次讲一下具体的操作配置。

## location配置

> location模块工作在虚拟主机server之下，对URL进行匹配，如果匹配成功就按照该location之中写的语句进行操作。

**语法** 

location [=|~|~*|^~] /uri/ { … }

**匹配规则**

|         模式        |         含义                        |
|:------------------|:---------------------------------: |
|location = /uri     | = 表示精确匹配，只有完全匹配上才能生效。  |
|location ^~ /uri|^~ 开头对URL路径进行前缀匹配，并且在正则之前。|
|location ~ pattern |区分大小写的正则匹配|
|location ~* pattern|不区分大小写的正则匹配|
|location /uri|不带任何修饰符，也表示前缀匹配，但是在正则匹配之后|
|location /|通用匹配，任何未匹配到其它location的请求都会匹配到，相当于switch中的default|

那么如果我们在一个虚拟主机下边写了很多location 规则，哪一个先匹配哪一个后匹配呢？
是这样的。
nginx会根据模糊程度排序的
- 首先精确匹配 =
- 其次前缀匹配 ^~
- 其次是按文件中顺序的正则匹配
- 然后匹配不带任何修饰的前缀匹配。
- 最后是交给 / 通用匹配
- 当有匹配成功时候，停止匹配，按当前匹配规则处理请求

## upstream
> 负载均衡模块

**语法**


## rewrite

> rewrite功能就是，使用nginx提供的全局变量或自己设置的变量，结合正则表达式和标志位实现url重写以及重定向。rewrite只能放在server{},location{},if{}中，并且只能对域名后边的除去传递的参数外的字符串起作用

**语法**
`rewrite regex replacement [flag];`

flag标志位
- last – 相当于Apache的[L]标记，表示完成rewrite，基本上都用这个 Flag
- break – 中止 Rewirte，不再继续匹配
- redirect – 返回临时重定向的 HTTP 状态 302
- permanent – 返回永久重定向的 HTTP 状态 301

last一般写在server和if中，而break一般使用在location中
last不终止重写后的url匹配，即新的url会再从server走一遍匹配流程，而break终止重写后的匹配





