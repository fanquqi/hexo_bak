---
title: nginx之请求限流限速问题
date: 2017-07-20 15:04:07
tags: nginx
---

# 单位时间按照IP地址限速 

工作当中经常需要按照IP地址限速，加白名单或者黑名单，这里面就会用到map来设置。看下边例子展示

```
http {  
    geo $white_ip {
        ranges;
        default 0;
        127.0.0.1-127.0.0.1 1;

        36.110.16.242-36.110.16.242 1
         
    }


    map $white_ip $white_ip_address {
        0 $binary_remote_addr;
        1 "";
    }

    limit_req_zone $white_ip_address zone=:10m rate=20r/s;
    
    ....

}
```

**解释下**

上述通过`geo模块` 设定了白名单 

**ngx_http_geo_module** 模块可以用来创建变量，其值依赖于客户端IP地址。
语法: geo [$address] $variable { ... }    设置在http 模块中  
`[$address]` 可以为空，使用默认变量也就是$remote_addr 其实例子中 `geo $white_ip {}`  就相当于`geo $remote_addr $white_ip {}`
`$white_ip` 命名为`white_ip` 
`ranges` 使用以地址段的形式定义地址，这个参数必须放在首位。为了加速装载地址库，地址应按升序定义。
`default` 设置默认值，如果客户端地址不能匹配任意一个定义的地址，nginx将使用此值。

注：如果36.110.16.242 这个IP地址访问本站, `white_ip` 这个变量值就是 1,否则就是0 


通过`map模块` 对$white_ip进行映射

**ngx_http_map_module** 模块可以创建变量，这些变量的值与另外的变量值相关联（上文的$white_ip）。允许分类或者同时映射多个值到多个不同值并储存到一个变量中，map指令用来创建变量，但是仅在变量被接受的时候执行视图映射操作，对于处理没有引用变量的请求时，这个模块并没有性能上的缺失。 

语法: map $var1 $var2 { ... }

上文map指令是将$white_ip值为0的，也就是受限制的IP，映射为客户端IP。将$white_ip值为1的，映射为空的字符串。
`limit_conn_zone`和`limit_req_zone`指令对于键为空值的将会被忽略，从而实现对于列出来的IP不做限制。

`limit_req_zone ` 真正操作限速 
**ngx_http_limit_req_module** 模块

# 下载限速

```
location /file { 
    limit_rate 128k;
# 限制下载速度128K/s 
  } 

# 如果想设置用户下载文件的前10m大小时不限速，大于10m后再以128kb/s限速可以增加以下配内容，修改nginx.conf文件

location /download { 
       limit_rate_after 10m; 
       limit_rate 128k; 
 }  
```

