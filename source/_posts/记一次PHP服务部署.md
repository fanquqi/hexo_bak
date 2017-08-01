---
title: 记一次PHP服务部署
date: 2017-07-05 18:58:03
tags: PHP
categories: 基础运维
---

> 本来很少接触这门世界上最好的语言，公司里面也没有，但是这次有这个需求，考虑到fastcgi与uwsgi有这么一点点共同点，我就照葫芦画瓢，打算用我们测试的nginx做转发。但是踩到几个坑，听我带着悔恨一点一点的说。。。

## nginx代理转发介绍
我们的测试跟线上的服务都是用nginx做全职代理转发
在nginx.conf 声明如下:
```
http {
...
include /usr/local/nginx/conf/servers/*/upstream.conf;
include /usr/local/nginx/conf/servers/*/site.conf;
...
}
```
所以就可以在server目录下建立各种监听二级域名的目录。

类似这种,`uwsgi`的转发使用`uwsgi_pass` ,`普通web代理`使用`proxy_pass`,`fastcgi`使用`fastcgi_pass`
下面举例uwsgi转发配置说明一下。
`/usr/local/nginx/conf/servers/dier/site.conf`
```
server {
    listen 443 ssl http2;
    server_name dier.chunyu.me;

    include /usr/local/nginx/conf/servers/common/ssl_config.location;
    location / {
        uwsgi_pass devops_uwsgi;
        include uwsgi_params;
    }
}


server {
    listen 80;
    server_name .chunyu.me;
# 强转https
    rewrite  ^/(.*)$  https://devops.chunyu.me/$1  permanent;
}
```

`/usr/local/nginx/conf/servers/dier/upstream.conf`
```
upstream devops_uwsgi {
    server 10.9.77.8:5001;
}
```

## PHP服务部署
> 源码编译安装，全程Google教程,直接按照参考地址配置即可。


PHP-FPM
PHP-FPM是一个PHP FastCGI管理器，是只用于PHP的,可以在 http://php-fpm.org/download下载得到。PHP-FPM其实是PHP源代码的一个补丁，旨在将FastCGI进程管理整合进PHP包中。必须将它patch到你的PHP源代码中，在编译安装PHP后才可以使用。FPM（FastCGI 进程管理器）用于替换 PHP-CGI 的大部分附加功能，对于高负载网站是非常有用的。它的功能包括：
1. 支持平滑停止/启动的高级进程管理功能；
2. 可以工作于不同的 uid/gid/chroot 环境下，并监听不同的端口和使用不同的 php.ini 配置文件（可取代 safe_mode 的设置）；
3. stdout 和 stderr 日志记录;
4. 在发生意外情况的时候能够重新启动并缓存被破坏的 opcode;
5. 文件上传优化支持;
6. “慢日志” – 记录脚本（不仅记录文件名，还记录 PHP backtrace 信息，可以使用 ptrace或者类似工具读取和分析远程进程的运行数据）运行所导致的异常缓慢;
7. fastcgi_finish_request() – 特殊功能：用于在请求完成和刷新数据后，继续在后台执行耗时的工作（录入视频转换、统计处理等）；
8.动态／静态子进程产生 ；
9. 基本 SAPI 运行状态信息（类似Apache的 mod_status）；
10. 基于 php.ini 的配置文件。


环境配置
```
yum -y install gcc gcc-c++
groupadd web
useradd -M -s /sbin/nologin -g web php
yum -y install epel-release
yum -y update
yum -y install libmcrypt libmcrypt-devel mcrypt mhash
yum -y install libxml2-devel libpng-devel libjpeg-devel zlib bzip2 bzip2-devel \
libtool-ltdl-devel pcre-devel openssl-devel freetype-devel libcurl-devel icu \
perl-libintl postgresql libicu-devel
```

下载解压
```
cd /usr/local/src/
wget http://cn2.php.net/distributions/php-5.6.27.tar.gz
tar -zxvf php-5.6.27.tar.gz
cd php-5.6.27/
```

编译安装
```
./configure \
--prefix=/usr/local/php5.6.27 \
--with-config-file-path=/usr/local/php5.6.27/etc/ \
--enable-inline-optimization \
--enable-shared \
--enable-opcache \
--enable-fpm \
--with-fpm-user=php \
--with-fpm-group=web \
--with-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-gettext \
--enable-mbstring \
--with-iconv \
--with-mcrypt \
--with-mhash \
--with-openssl \
--enable-bcmath \
--enable-soap \
--with-libxml-dir \
--enable-pcntl \
--enable-shmop \
--enable-sysvmsg \
--enable-sysvsem \
--enable-sysvshm \
--enable-sockets \
--enable-intl \
--with-curl \
--with-zlib \
--enable-zip \
--with-bz2 \
--enable-xml \
--with-pcre-dir \
--with-gd \
--enable-static \
--enable-wddx \
--with-xmlrpc \
--with-libdir=/usr/lib64 \
--with-jpeg-dir=/usr/lib64 \
--with-freetype-dir=/usr/lib64 \
--with-png-dir=/usr/lib64
```
```
make && make install
```

简单配置
```
cp php.ini-development /usr/local/php5.6.27/etc/php.ini
cp /usr/local/php5.6.27/etc/php-fpm.conf.default /usr/local/php5.6.27/etc/php-fpm.conf
```

创建开机启动
```
vi /lib/systemd/system/php-fpmd.service
```
```
[Unit]
Description=The PHP FastCGI Process Manager
After=network.target

[Service]
Type=forking
PIDFile=/run/php-fpm.pid
ExecStart=/usr/local/php5.6.27/sbin/php-fpm --daemonize -g /run/php-fpm.pid
ExecReload=/bin/kill -USR2 $MAINPID
ExecStop=/bin/kill -SIGINT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
```
systemctl enable php-fpmd.service
systemctl start php-fpmd.service
```


`注意` 

**php.ini** 中设置`open_basedir=/usr/local/nginx/html/webapps`
**php-frm.conf** 中`security.limit_extensions = .php .html .js .css .jpg .jpeg .gif .png .htm .txt` 


## mysql 安装

> ansible 自动安装，脚本以后会附上，导入数据手动，没有任何问题

## nginx 配置 
> 我最开始的想法是，在起PHP这个服务的云主机上起一个nginx 做web服务器。但是我组大神告诉我，不用这么麻烦直接用测试服nginx代理就好。于是我没有反驳，毕竟这个看起来更简单更合理。于是我就开始配置。。。

最开始配置,如下

```
server {
    listen 80;
    server_name dier.chunyu.me;

    location ~ [^.]+\.php$ {
        root   /usr/share/webapps;
        fastcgi_pass 10.0.0.1:9000;
        index index.html index.php;
        #fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include fastcgi_params;

    }
}
```
写到一半，我突然发现静态文件怎么办，于是我直接把`location ~ [^.]+\.php$`` 改成了 `location /` 所有文件都这么走。
结果问题就出现了，如下图。css文件的请求头`Content_type`为`text/html` 

![](http://or2jd66dq.bkt.clouddn.com/css_error.png)
我试着在测试服nginx上各种`add_header` 都不好使，于是请教之前大神，他一脸不屑的看着我，看了三秒。。。然后他解决，我看到机器上文件的修改，一次次add_header ,过了20多分钟，他扭头给我说，这个fastcgi好像不支持静态文件代理，而且他的代码里面没有加判断。你在这个机器上装个nginx吧。。。
恩，于是我有用ansible跑了一遍安装nginx的脚本。
配置如下:

```
location ~ [^.]+\.php$ {
    root           /usr/local/nginx/html/webapps;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    #fastcgi_param  SCRIPT_FILENAME  /usr/local/nginx/html/webapps$fastcgi_script_name;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include        fastcgi.conf;
 }

location ~* \.(css|js|png|jpg|jpeg|gif|ico)$ {
    expires max;
    log_not_found off;
 }

```
问题解决。。。


## 总结

- 多学习，多看书，少说话
- 如果说话学会和人一样说话
- 多做总结，比如写博客加深一下印象，不太懂也没有关系，写着写着有可能就懂了。
 
## 一台机器上启动两个PHP服务

> SEO的哥们找我说克隆一个和之前一模一样的服务，我的原则是PHP这种漏洞比较多的服务最好还是给他们独立出来不要跟线上有联系，前不久让一个白帽子给我们扫了一下，我们才发现之前商务部门归到我们这边的一个提供PHP服务的机器直接被人拿到了root的shell，真可怕。于是，我这次把他们的数据库都从线上拆出来了。放在本地。

**方法**

直接再装一个php-fpm （我刚开始用一个PHP-fpm提供和两个PHP服务的动态处理在nginx中把他们的静态文件分开，事实证明不可以，会发生一些奇怪的情况，两个系统的各种配置，包括数据库都会混淆）换个端口启动就好了

参考地址](https://my.oschina.net/yule526751/blog/795807)



