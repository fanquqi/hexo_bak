---
title: Logstash优化
date: 2017-06-15 15:59:39
tags: ELK, logstash
categories: 基础运维
---

> 之前部署的ELK只加了nginx_access, elapsed两个日志，每天的日志量有400G左右，架构可以撑住。但是现在又引入了一些新的日志，并且随着SEO部门的来临。每天晚上服务都被爬虫爬。日志量也翻了两三翻。产生了日志延时的问题。有的延时能有几个小时。
![](http://or2jd66dq.bkt.clouddn.com/kibana%E5%BB%B6%E6%97%B6.png)
看kibana中上传的时间跟日志里的时间差距好几个小时。说明日志已经阻塞了。

## 问题分析

这期间机器负载啥的一直也没有报警，ES集群也很健康，所以初步估计是logstash遇到了瓶颈。因为我们有grok，之前就听说这个过滤会对速度影响很大，还有geoip等。

## 解决办法

我第一想到的是，收集日志换成`filebeat`然后像之前加的syslog一样放到kafka缓冲一下，再用logstash插入到ES，但是无奈，机器资源有限，kafka又是直接写问加你系统，运维机的磁盘已经很满了，在扩容磁盘之前是无望了。所以现在只能优化一下logstash，坚持到磁盘扩容。

## logstash优化

- 加大内存
- 增加线程
- 减少grok或者去掉geoip定位

另外更改@timestamp 参考:http://udn.yyuap.com/doc/logstash-best-practice-cn/filter/date.html
为了防止每次延时比较大的时候图像显示不准确。
