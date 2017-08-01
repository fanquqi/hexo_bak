---
title: nginx之安装配置
date: 2017-07-06 19:52:30
tags: nginx
categories: 基础运维    
---

nginx 相关知识参考网站
[Tengine](http://tengine.taobao.org/)

> nginx 首先从安装配置说起问题说起

# 安装
> 一般都是源码编译安装，直接解压编译安装就好没有啥可说的 一般需要安装PCRE zlib openssl 库 以及所需要模块例如openLDAP等

**特别说明** 
安装完成之后再添加模块,需要重启服务，reload不会生效。 
[参考链接](http://taokey.blog.51cto.com/4633273/1318719)

# 配置
> nginx的配置是一门很深的功课，首先我们要对基本的http协议特别了解，配置过程可能要各种rewrite，各种location。不要慌，一点一点来。

## 配置文件参数详解
> 首先对`nginx.conf`里面的各个配置项进行一下解释。 

首先说下少数几个高级配置，一般写在开头，模块配置之上
**进程运行的用户组** 
``` 
user  nginx nginx;
```

**进程数** 一般跟CPU核数相匹配。nginx启动后有多少个worker处理请求，不包括master，(master不处理请求，二十主要接受客户端的请求并分配给worker处理)这里还涉及到下边要说的`worker_connections`, 正常被大家接受的nginx最大连接数就是靠这两个计算出来的.

- nginx作为http服务器的时候：
    - max_clients = worker_processes * worker_connections
- nginx作为反向代理服务器的时候：
    - max_clients = worker_processes * worker_connections/4

具体为什么去[参考](http://liuqunying.blog.51cto.com/3984207/1420556)
```
worker_processes  8;
```

**最大打开文件数量** 这个如果没有设置会使用linux系统默认的文件最大打开数 `ulimit -a`可以查看到。
```
worker_rlimit_nofile 65535;
```

### Events模块

这里包含nginx所有处理连接的设置
```
events {
    worker_connections 2048;

    use epoll;
}
```
`worker_connections`表示一个worker同时打开最大连接数。
`use epoll`定义轮询方法 如果你的内核为linux 2.6+ 应该使用epoll异步非阻塞模型

### http模块

> HTTP模块控制着nginx http处理的所有核心特性

```

http {
 
    server_tokens off;
 
    sendfile on;

    tcp_nopush on;
    tcp_nodelay on;

    ...

}
```

`server_tokens` 这个只是在错误页面不显示nginx版本，为了安全。
`sendfile` ,`tcp_nopush`,`tcp_nodelay`这三条一般同时出现 提高读写速度 提升性能 ，tcp_nopush 依赖sendfile, tcp_nodelay 不缓存数据，(禁用nagle算法，发送小数据不缓存直接发)


```
log_format main '$remote_addr - - [$time_local] "$request" $status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" [$request_time, $upstream_response_time] $host ($remote_port) "sid=$cookie_sessionid"';
access_log  logs/access.log  main;
```
定义日志格式 `main` 下边引用该格式。其实可以设置成json格式的，以后收集解析也方便。

```
lingering_close off;
keepalive_timeout 5;
send_timeout 20;
proxy_connect_timeout 30;
proxy_read_timeout 20;
proxy_send_timeout 20;

```

`lingering_close` 定义关闭连接的方式，有三个选项 **off|on|always** `off`：请求完成之后，关闭连接，不管此时有没有收到客户端数据；`on`是中间值，一般情况下在关闭连接前都会处理连接上的用户发送的数据，除了有些情况下在业务上认定这之后的数据是不必要的；`always`无条件处理完所有用户请求。Tengine 默认off效率高些，但是存在误杀状况，nginx默认on 算个小坑。

`keepalive_timeout` 给客户端分配keep-alive链接超时时间

`send_timeout` 发送响应的超时时间，两个客户端请求之间的时间

这几个一般用在nginx做反向代理的时候
`proxy_connect_timeout` 后端服务器连接的超时时间_发起握手等候响应超时时间
`proxy_read_timeout` 后端服务器处理请求的时间
`proxy_send_timeout` 后端服务器数据回传时间_就是在规定时间之内后端服务器必须传完所有的数据

```

include /etc/nginx/mime.types;
default_type application/octet-stream;
```
include只是一个在当前文件中包含另一个文件内容的指令。这里我们使用它来加载稍后会用到的一系列的MIME类型。其实就是content-type与扩展名的映射。在客户端发来一个请求之后，nginx通过扩展名找到对应的content-type,下载返回的头信息中，浏览器收到之后会按照这个类型做解析展示。这样就不至于发生css文件本当做html一样当文本展示了。如果在mime.types中没有找到，会使用default_type


```
## GZIP Setting
gzip  on;
gzip_min_length  1000;
gzip_buffers     4 8k;
gzip_http_version  1.0;
gzip_comp_level  5;
gzip_types       text/plain text/css application/x-javascript application/json application/xml;
```
gzip是GNU zip的缩写，它是一个GNU自由软件的文件压缩程序，可以极大的加速网站.有时压缩比率高到80%,近来测试了一下,最少都有40%以上,还是相当不错的。
`gzip`
决定是否开启gzip模块

`gzip_min_length`
当返回内容大于此值时才会使用gzip进行压缩,以K为单位,当值为0时，所有页面都进行压缩
`gzip_buffers`
设置gzip申请内存的大小,其作用是按块大小的倍数申请内存空间
`gzip_http_version`
用于识别http协议的版本，早期的浏览器不支持gzip压缩，用户会看到乱码，所以为了支持前期版本加了此选项,目前此项基本可以忽略
`gzip_comp_level`
设置gzip压缩等级，等级越底压缩速度越快文件压缩比越小，反之速度越慢文件压缩比越大
`gzip_types`
设置需要压缩的MIME类型,非设置值不进行压缩
## server模块

> server模块是http的子模块，定义虚拟主机
格式
```
server {
    listen      80;
    server_name map.baidu.com www.baidu.com; 
    client_max_body_size 10m;
    root   /Users/yangyi/www;
    index  index.php index.html index.htm; 
    charset utf-8;

    }
}
```

`server {}` server标志虚拟主机开始，在 {} 中配置
`listen `监听 80端口
`server_name`用来指定IP地址或者域名，多个域名之间用空格分开。这里指定域名为map.baidu.com 或者www.baidu.com。
`client_max_body_size`  文件上传大小
`charset` 声明网站默认编码格式

直接转发到到10.0.0.1:9000，这几个proxy_set_header的意思是改变请求头的Host为客户端的Host，ip 否则在下层的服务端会认为客户端是这台代理的nginx。
X-Forwarded-For 是一个 HTTP 扩展头部。HTTP/1.1（RFC 2616）协议并没有对它的定义，它最开始是由 Squid 这个缓存代理软件引入，用来表示 HTTP 请求端真实 IP。如今它已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用，并被写入 RFC 7239（Forwarded HTTP Extension）标准之中。
格式如下
> X-Forwarded-For: client, proxy1, proxy2

可以看到，XFF 的内容由「英文逗号 + 空格」隔开的多个部分组成，最开始的是离服务端最远的设备 IP，然后是每一级代理设备的 IP。
如果一个 HTTP 请求到达服务器之前，经过了三个代理 Proxy1、Proxy2、Proxy3，IP 分别为 IP1、IP2、IP3，用户真实 IP 为 IP0，那么按照 XFF 标准，服务端最终会收到以下信息：

> X-Forwarded-For: IP0, IP1, IP2

这个在多级代理的时候可以设置下，逻辑关系更清晰。

## location 模块

> 在server内部

```
location / {
            root   /Users/yangyi/www;
            index  index.php index.html index.htm;
        }
```

```
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_pass http://10.0.0.1:9000;
    proxy_redirect default;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```


**一个配置样例**
```
user www-data;

pid /var/run/nginx.pid;

worker_processes auto;

worker_rlimit_nofile 100000;

 
events {

    worker_connections 2048;
    multi_accept on;
    use epoll;

}

http {
    server_tokens off;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    log_format main '$remote_addr - - [$time_local] "$request" $status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" [$request_time, $upstream_response_time] $host ($remote_port) "sid=$cookie_sessionid"';
    access_log  logs/access.log  main;

    error_log /var/log/nginx/error.log crit;
 

    keepalive_timeout 10;

    client_header_timeout 10;

    client_body_timeout 10;
    reset_timedout_connection on;

    send_timeout 10;
    limit_conn_zone $binary_remote_addr zone=addr:5m;
    limit_conn addr 100;
    include /etc/nginx/mime.types;
    default_type text/html;
    charset UTF-8;
    gzip on;
    gzip_disable "msie6";
    gzip_proxied any;
    gzip_min_length 1000;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    open_file_cache max=100000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;

}


最后 关于nginx和http https等的设置可以参考[博客](https://imququ.com/)
