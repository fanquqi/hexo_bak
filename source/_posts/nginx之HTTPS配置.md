---
title: nginx之HTTPS配置
date: 2017-07-07 15:15:38
tags: nginx
categories: 基础运维
---

> https现在是标配了，HTTPS 可以给用户带来更安全、比如减少了被劫持的概率，更好隐私保护的网络体验，这些好处大家都耳熟能详，本文不再赘述。现在很多浏览器都在推行HTTPS的普及。尽快升级吧。

其实大体就是分为`ssl证书申请`和`配置HTTPS`两个步骤

## 证书申请
这次介绍并没有从申请证书开始，因为之前已经申请过了，申请步骤请[参考](https://aotu.io/notes/2016/08/16/nginx-https/index.html)
SSL 证书主要有两个功能：加密和身份证明，通常需要购买，也有免费的，通过第三方 SSL 证书机构颁发。分为企业级别和个人级别。
SSL 具体加密实现[参考](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
查看证书相关配置，包括哪个机构颁发的，过期时间等等信息可以到https://www.chinassl.net 自助查看

## nginx HTTPS配置

首先我们把得到的domain.key domain.crt 放到 nginx的conf下，
可以在nginx.conf 的`server`中配置

**基础配置**
```
 server {
     listen              443 ssl http2;
     #证书文件(注意路径及权限)
     ssl on;
     ssl_certificate     example.com.crt;
     #私钥文件
     ssl_certificate_key example.com.key;
     ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
     ssl_ciphers         HIGH:!aNULL:!MD5;

#....
}
```
必须使用监听命令 listen 的 ssl 参数和定义服务器证书文件和私钥文件

`ssl_protocols` 可以用来限制连接只包含 SSL/TLS 的加強版本，默认值如上。
`ssl_ciphers` 选择加密套件，不同的浏览器所支持的套件（和顺序）可能会不同。这里指定的是OpenSSL库能够识别的写法，你可以通过 openssl -v cipher 'RC4:HIGH:!aNULL:!MD5'（后面是你所指定的套件加密算法） 来看所支持算法。



**加强 HTTPS 安全性**

```
ssl_prefer_server_ciphers ON;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-Xss-Protection 1;

```

`ssl_prefer_server_ciphers ON`设置协商加密算法时，优先使用我们服务端的加密套件，而不是客户端浏览器的加密套件。
`add_header X-Frame-Options DENY`减少点击劫持



**HTTPS优化参数**
```
ssl_session_cache shared:SSL:10m;

ssl_session_timeout 10m;
ssl_buffer_size 1400;

```

`ssl_session_cache shared:SSL:10m;` 设置ssl/tls会话缓存的类型和大小。如果设置了这个参数一般是shared，buildin可能会参数内存碎片，默认是none，和off差不多，停用缓存。如shared:SSL:10m表示我所有的nginx工作进程共享ssl会话缓存，官网介绍说1M可以存放约4000个sessions。 
`ssl_session_timeout 10m;`  客户端可以重用会话缓存中ssl参数的过期时间。
`ssl_buffer_size 1400;` 缓冲区调优，从1.5.9版本开始,Nginx允许使用ssl_buffer_size指令自定义TLS缓冲区的大小，默认值是 16 KB,但是这个值不一定是最优化的,尤其是你希望首字节数据被尽早发送时,有报告显示使 用1400字节的配置可以显著减少延迟。[参考](http://fangpeishi.com/optimizing-tls-record-size.html)


**配置参考**
```
ssl                  on;
ssl_session_timeout  30m;
ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;

ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA:!CAMELLIA;

ssl_prefer_server_ciphers   on;
ssl_buffer_size 1400;

ssl_session_cache    shared:SSL:10m;
ssl_certificate      domain.crt;
ssl_certificate_key  domain.key;

```
