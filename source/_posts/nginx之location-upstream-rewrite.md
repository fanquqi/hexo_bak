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

&emsp;&emsp;那么如果我们在一个虚拟主机下边写了很多location 规则，哪一个先匹配哪一个后匹配呢？
是这样的。
nginx会根据模糊程度排序的

- 首先精确匹配 =
- 其次前缀匹配 ^~
- 其次是按文件中顺序的正则匹配
- 然后匹配不带任何修饰的前缀匹配。
- 最后是交给 / 通用匹配
- 当有匹配成功时候，停止匹配，按当前匹配规则处理请求

## upstream
> 负载均衡模块,负责根据配置合理分流到各个代理节点，而且自带后端节点健康检查（需要自己通过proxy_read_timeout指令和proxy_next_upstream指令配置）

[参考文章](http://nolinux.blog.51cto.com/4824967/1594029)
**语法** `upstream name {server ip:port 状态}`

```
http {
    upstream name {
        
        [ip_hash;]
        server ip:port [weight=n];
        server ip:port [weight=n];
    }



server {
    listen 80;
    server_name   www.xxx.com; 

    proxy_pass http://name;
    或者
    uwsgi_pass 
    或者
    fast_cgi_pass
    }
    
}
```
上下文 http server location 



负载均衡策略: 
- 轮询(默认)
- 设置权重(weight)
- ip_hash(每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。)

其次还可以定义状态

- down 表示单前的server暂时不参与负载
- weight 默认为1.weight越大，负载的权重就越大。
- max_fails ：允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream 模块定义的错误
- fail_timeout:max_fails次失败后，暂停的时间。
- backup： 其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻。

### 负载均衡后端节点健康检查

&emsp;&emsp;严格来说，nginx自带是没有针对负载均衡后端节点的健康检查的，但是可以通过默认自带的ngx_http_proxy_module.
  模块和ngx_http_upstream_module模块中的相关指令来完成当后端节点出现故障时，自动切换到健康节点来提供访问。


这里学习下ngx_http_proxy_module 模块中的 `proxy_connect_timeout` 指令、`proxy_read_timeout`指令和`proxy_next_upstream`指令

**设置与后端服务器建立连接的超时时间。** 应该注意这个超时一般不可能大于75秒。
    
    proxy_connect_timeout 60s;

**设置从后端服务器读取响应的超时**

    proxy_read_timeout 60s;

**指定在何种情况下一个失败的请求应该被发送到下一台后端服务节点**

    proxy_next_upstream error timeout;

`error`      和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现错误

`timeout`    和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现超时

`invalid_header`  后端服务器返回空响应或者非法响应头

`http_500`    后端服务器返回的响应状态码为500

`http_502`    后端服务器返回的响应状态码为502

`http_503`    后端服务器返回的响应状态码为503

`http_504`    后端服务器返回的响应状态码为504

`http_404`    后端服务器返回的响应状态码为404

`off`         停止将请求发送给下一台后端服务器

&emsp;&emsp;还可以通过tengine来实现，淘宝技术团队开发的`nginx_upstream_check_module`模块来实现。
如果没有使用tengine需要打补丁。
配置起来比上述方法简单一些。[参考地址](http://tengine.taobao.org/document_cn/http_upstream_check_cn.html)

```
http {
    upstream cluster1 {
        # simple round-robin
        server 192.168.0.1:80;
        server 192.168.0.2:80;
        check interval=3000 rise=2 fall=5 timeout=1000 type=http;
        check_http_send "HEAD / HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx;
    }
    upstream cluster2 {
        # simple round-robin
        server 192.168.0.3:80;
        server 192.168.0.4:80;
        check interval=3000 rise=2 fall=5 timeout=1000 type=http;
        check_keepalive_requests 100;
        check_http_send "HEAD / HTTP/1.1\r\nConnection: keep-alive\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx;
```

&emsp;&emsp;上面配置的意思是，对name这个负载均衡条目中的所有节点，每个3秒检测一次，请求2次正常则标记 realserver状态为up，如果检测 5 次都失败，则标记 realserver的状态为down，超时时间为1秒。

- interval：向后端发送的健康检查包的间隔。
- fall(fall_count): 如果连续失败次数达到fall_count，服务器就被认为是down。
- rise(rise_count): 如果连续成功次数达到rise_count，服务器就被认为是up。
- timeout: 后端健康请求的超时时间。
- type：健康检查包的类型，现在支持以下多种类型
    - tcp：简单的tcp连接，如果连接成功，就说明后端正常。
    - ssl_hello：发送一个初始的SSL hello包并接受服务器的SSL hello包。
    - http：发送HTTP请求，通过后端的回复包的状态来判断后端是否存活。
    - mysql: 向mysql服务器连接，通过接收服务器的greeting包来判断后端是否存活。
    - ajp：向后端发送AJP协议的Cping包，通过接收Cpong包来判断后端是否存活。

**check_keepalive_requests**

&emsp;&emsp;该指令可以配置一个连接发送的请求数，其默认值为1，表示Tengine完成1次请求后即关闭连接。

**check_http_send**

&emsp;&emsp;该指令可以配置http健康检查包发送的请求内容。为了减少传输数据量，推荐采用"HEAD"方法。

当采用长连接进行健康检查时，需在该指令中添加keep-alive请求头，如："HEAD / HTTP/1.1\r\nConnection: keep-alive\r\n\r\n"。
同时，在采用"GET"方法的情况下，请求uri的size不宜过大，确保可以在1个interval内传输完成，否则会被健康检查模块视为后端服务器或网络异常。

**check_http_expect_alive**


&emsp;&emsp;该指令指定HTTP回复的成功状态，默认认为2XX和3XX的状态是健康的。如果爬虫多404多可以把4XX也写进去。

## rewrite

> rewrite功能就是，使用nginx提供的全局变量或自己设置的变量，结合正则表达式和标志位实现url重写以及重定向。rewrite只能放在server{},location{},if{}中，并且只能对域名后边的除去传递的参数外的字符串起作用

**语法**
`rewrite regex replacement [flag];`

flag标志位

- last – 相当于Apache的[L]标记，表示完成rewrite，浏览器地址不变
- break – 中止 Rewirte，不再继续匹配，浏览器地址不变
- redirect – 返回临时重定向的 HTTP 状态 302 浏览器地址显示跳转后的地址
- permanent – 返回永久重定向的 HTTP 状态 301 浏览器地址显示跳转后的地址

last一般写在server和if中，而break一般使用在location中

last不终止重写后的url匹配，即新的url会再从server走一遍匹配流程，而break终止重写后的匹配。

**简单举例**

```
// 访问/example.html 的时候重写到/index.html
rewrite /example.html /index.html last;
// 访问/example.html 的时候重写到/index.html,并停止匹配
rewrite /example.html /index.html break;
// 把 /search/key => /search.html?keyword=key
rewrite '^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$' /data?file=$3.$4 last;
 "^" 表示开头匹配 "$" 表示结尾匹配 而且表示路径的"/"是需要转义的，"$1"表示表达式匹配到的第一个括号里面的内容，即([a-z]{2})
```

正则实例可以[参考](http://www.cnblogs.com/zxin/archive/2013/01/26/2877765.html)


