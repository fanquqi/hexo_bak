---
title: uwsgi笔记
date: 2017-07-13 14:38:09
tags: uwsgi
categories: 基础运维
---

> 前言，来公司一年了，对uwsgi的操作，只停留在部署，或者restart，并没有主动配置过，今天看nginx的时候看到nginx(我们的nginx是只转发)的请求是转发的本地某个端口，是个uwsgi提供的fastrouter服务。在此记录一下。

由于我们后端使用的都是Django项目，所以整体都是这个架构`nginx`-->`uwsgi`-->`django`.所以对uwsgi的了解也很重要。
fastrouter使用放到后面先说，uwsgi的普通配置。

版本：uWSGI==2.0.12

## 简介

WSGI（Python Web Server Gateway Interface，缩写为WSGI） 是一种 Web 服务器网关接口。它是一个 Web 服务器（如 Nginx）与应用服务器（如 uWSGI 服务器）通信的一种规范。
uwsgi 是一种协议
uWSGI uWSGI是一个Web服务器，它实现了WSGI协议、uwsgi、http等协议。uWSGI，既不用wsgi协议也不用FastCGI协议，而是自创了一个uwsgi的协议，uwsgi协议是一个uWSGI服务器自有的协议，它用于定义传输信息的类型（type of information），每一个uwsgi packet前4byte为传输信息类型描述，它与WSGI相比是两样东西。据说该协议大约是fcgi协议的10倍那么快。

优点：
- 超快的性能。
- 低内存占用（实测为apache2的mod_wsgi的一半左右）。
- 多app管理。
- 详尽的日志功能（可以用来分析app性能和瓶颈）。
- 高度可定制（内存大小限制，服务一定次数后重启等）。

版本：uWSGI==2.0.12

## 安装配置

安装就是直接在virtualenv中pip安装就好了
pip install uWSGI
 



## uwsgitop

> 监控工具

### 部署

pip install uwsgitop 

### 使用
uwsgitop /tmp/stats.sock


## plugins使用

### fastrouter
这是一个负载均衡插件，比如说四个uwsgi节点提供服务，这样在nginx上面可以配置成upstream，分流到四个uwsgi服务上，但是如果是上线，或者有一个节点挂掉了怎么办，只能是收到500报警在手动剔除吗？ to yung to sample!! 当然不是，这个时候就用到fastrouter了。
看[官网](http://uwsgi-docs-cn.readthedocs.io/zh_CN/latest/Fastrouter.html?highlight=fast)

> For advanced setups uWSGI includes the “fastrouter” plugin, a proxy/load-balancer/router speaking the uwsgi protocol. It is built in by default. You can put it between your webserver and real uWSGI instances to have more control over the routing of HTTP requests to your application servers.

它的功能proxy/load-banlance/router
简述一下配置：

**nginx配置**
```
location /test {
        include    uwsgi_params;
        uwsgi_pass 127.0.0.1:3030;
    }
```

**fastrouter-server端配置**
```
<uwsgi id = "fastrouter">
    <fastrouter>127.0.0.1:3030</fastrouter>
    <fastrouter-subscription-server>127.0.0.1:3131</fastrouter-subscription-server>
    <enable-threads/>
    <master/>
    <fastrouter-stats>127.0.0.1:9595</fastrouter-stats>
</uwsgi>
```

 - 3030为当前uWSGI fastrouter server的端口，前面的空代表当前主机地址。（nginx会用到这个端口）
 - 3131fastrouter-subscription-server 表示当前uWSGI fastrouter server的订阅地址。（web应用服务会用到）
 - stats：uWSGI的统计服务机制，访问会返回一个json对象，都是[状态统计信息](http://uwsgi-docs.readthedocs.io/en/latest/StatsServer.html) 格式可以是一个端口，也可以是一个socket

**uwsgi实例配置**

实例1
```
<uwsgi id = "subserver1">
    <stats>127.0.0.1:9393</stats>
    <processes>4</processes>
    <enable-threads/>
    <memory-report/>
    <subscribe-to>127.0.0.1:3131:test</subscribe-to>
    <socket>127.0.0.1:3232</socket>
    <file>./server.py</file>
    <master/>
    <weight>8</weight>
</uwsgi>
```
实例2
```
<uwsgi id = "subserver2">
    <stats>127.0.0.1:9494</stats>
    <processes>4</processes>
    <enable-threads/>
    <memory-report/>
    <subscribe-to>127.0.0.1:3131:test</subscribe-to>
    <socket>127.0.0.1:3333</socket>
    <file>./server.py</file>
    <master/>
    <weight>2</weight>
</uwsgi>
```

- <stats>127.0.0.1:9494</stats> 这个可以把9494改为0 则为自动分配 也可以改成socket
- 我们通过subscribe-to变量来订阅fastrouter server（127.0.0.1:3131）,冒号后跟着的是对应请求的域名，只有来自当前域名的请求才会进入当前web节点。当然这个可以设置多个subscribe-to，例如：subscribe-to=127.0.0.1:3131:test1

- weight 权重分配

由于我们线上使用的是fastrouter,所以在这儿就只说了这个，其实uwsgi的负载均衡使用不只有这一种手段
有兴趣可以看下[这篇博文](http://www.cnblogs.com/codeape/p/4064815.html)

### harakiri


