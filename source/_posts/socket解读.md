---
title: socket解读
date: 2017-07-18 10:24:52
tags: socket
categories: 基础运维
---

# 前言
> 之前一直部署各种开源软件，包括很多RPC服务，其中很多都会有一个socket，但是很多地方可以用地址加端口替代，我虽然说做运维也将近两年了，但是毕竟大学是数学专业，不是计算机科班出身，对什么TCP/IP啊socket啊，都得是自己看书了解。so，抽个空补习了一下。

用廖雪峰老师的话说
> Socket是网络编程的一个抽象概念。通常我们用一个Socket表示“打开了一个网络链接”，而打开一个Socket需要知道目标计算机的IP地址和端口号，再指定协议类型即可。

# TCP/IP
要理解socket首先要理解TCP/IP，面试过程当中我们经常被问到ISO七层模型相关的知识，类似TCP/IP工作在第几层？
TCP/IP（Transmission Control Protocol/Internet Protocol）即传输控制协议/网间协议，定义了主机如何连入因特网及数据如何再它们之间传输的标准，

从字面意思来看TCP/IP是TCP和IP协议的合称，但实际上TCP/IP协议是指因特网整个TCP/IP协议簇。不同于ISO模型的七个分层，TCP/IP协议参考模型把所有的TCP/IP系列协议归类到四个抽象层中

应用层：TFTP，HTTP，SNMP，FTP，SMTP，DNS，Telnet 等等

传输层：TCP，UDP

网络层：IP，ICMP，OSPF，EIGRP，IGMP

数据链路层：SLIP，CSLIP，PPP，MTU

每一层都是建立在下一层提供的服务上，为上一层提供服务



python socket客户端编写(down 新浪首页)
```
# -*- coding:utf-8 -*-
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

s.connect(('www.sina.com',80))

s.send('GET / HTTP/1.1\r\nHost: www.sina.com.cn\r\nConnection: close\r\n\r\n')

buffer = []
while True:

    d = s.recv(2048)
    if d:
        buffer.append(d)
    else:
        break

data=''.join(buffer)
s.close
header, html = data.split('\r\n\r\n',1)
print header
```

server端编写
