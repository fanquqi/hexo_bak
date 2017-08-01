---
title: curl查看接口各阶段响应时间
date: 2017-06-16 12:02:57
tags: curl
categories: 基础运维
---
### **使用场景**
--------

> 有接口查询比较慢，需要查找具体到底是哪块响应时间比较长，然后再定位具体的问题。

curl 有个 -w参数。可以格式化输出我们可以使用。
它能够按照指定的格式打印某些信息，里面可以使用某些特定的变量，而且支持 \n、\t和 \r 转义字符。提供的变量很多，比如 status_code、local_port、size_download 等等，这篇文章我们只关注和请求时间有关的变量（以 time_ 开头的变量）。

**具体用法**

``` 
➜  ~ curl 'https://api.mch.weixin.qq.com/pay/unifiedorder' -w '\ntime_namelookup:  %{time_namelookup}\ntime_connect:  %{time_connect}\ntime_appconnect:  %{time_appconnect}\ntime_pretransfer:  %{time_pretransfer}\ntime_redirect:  %{time_redirect}\ntime_starttransfer:  %{time_starttransfer}\ntotal:  %{time_total}\n'
<xml><return_code><![CDATA[FAIL]]></return_code>
<return_msg><![CDATA[请使用post方法]]></return_msg>
</xml>
time_namelookup:  0.013
time_connect:  0.018
time_appconnect:  0.113
time_pretransfer:  0.113
time_redirect:  0.000
time_starttransfer:  0.180
total:  0.180
```

具体变量解读：

`time_namelookup`：DNS 域名解析的时候，就是把 URL 转换成 ip 地址的过程
`time_connect`：TCP 连接建立的时间，就是三次握手的时间
`time_appconnect`：SSL/SSH 等上层协议建立连接的时间，比如 `connect/handshake` 的时间
`time_redirect`：从开始到最后一个请求事务的时间
`time_pretransfer`：从请求开始到响应开始传输的时间
`time_starttransfer`：从请求开始到第一个字节将要传输的时间
`time_total`：这次请求花费的全部时间

而我们这个接口使用的是openvpn访问的外网，openvpn使用的openssl认证。所以基本定位到问题就在openvpn的ssl认证过程中。


