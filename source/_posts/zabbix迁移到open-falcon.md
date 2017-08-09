---
title: zabbix迁移到open-falcon
date: 2017-06-06 17:53:38
tags: falcon, 监控
categories: 监控
---
# falcon 介绍
>由于zabbix监控的可拓展性不是很高，业务的监控并不是很灵活，所以运维打算换成灵活度更好的open-falcon，他是小米公司推出的一款开源软件，基于go语言开发，安装比较方便。因为有着比较好看及灵活UI所以我们使用起来也是十分方便。但是社区相比zabbix小很多，坑也多一点，毕竟第一个吃螃蟹的人得付出点勇气。

![](http://or2jd66dq.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-06%20%E4%B8%8B%E5%8D%885.57.09.png)

## 优点说明

* 水平扩展能力：支持每个周期上亿次的数据采集、告警判定、历史数据存储和查询

* 高效率的告警策略管理：高效的portal、支持策略模板、模板继承和覆盖、多种告警方式、支持callback调用

* 人性化的告警设置：最大告警次数、告警级别、告警恢复通知、告警暂停、不同时段不同阈值

* 高效率的graph组件：单机支撑200万metric的上报、归档、存储（周期为1分钟）

* 高效的历史数据query组件：采用rrdtool的数据归档策略，秒级返回上百个metric一年的历史数据

* dashboard：多维度的数据展示，用户自定义Screen

* 高可用：整个系统无核心单点，易运维，易部署，可水平扩展

* 开发语言： 整个系统的后端，全部golang编写，portal和dashboard使用python编写。

## 组件介绍

### **agent 组件**

    * 监控客户端 相当于zabbix_agent 收集监控数据，提供接口供我们写脚本上传数据
    * 支持自发现，默认自带很多基本监控，它的自定义监控指标通过HBS（HBS读取portal数据库）获得
    * 每隔60秒push给Transfer。agent与Transfer建立了长连接，数据发送速度比较快，agent提供了一个http接口/v1/push用于接收用户手工push的一些数据，然后通过长连接迅速转发给Transfer。

### **Aggregater组件**   集群聚合模块。（目前我们还没有应用）

    * 聚合某集群下的所有机器的某个指标的值，比如所有机器的qps加和才是整个集群的qps，所有机器的request_fail数量 ÷ 所有机器的request_total数量=整个集群的请求失败率然后push回监控server端。

### **hbs**（heartbeat Server）组件
     心跳服务器，公司所有agent都会连到HBS，每分钟发一次心跳请求。
     agent发送心跳信息给HBS的时候，会把hostname、ip、agent version、plugin version等信息告诉HBS，HBS负责更新host表。 插入到portal数据库中
 
### **portal组件**
     Portal是用来配置报警策略的，策略存放于数据库中，通过hbs的读取传递到agent端

### **transfer组件**
     transfer是数据转发服务。它接收agent上报的数据，然后按照哈希规则进行数据分片、并将分片后的数据分别push给graph&judge等组件。

### **judge组件**
     Judge用于告警判断，agent将数据push给Transfer，Transfer不但会转发给Graph组件来绘图，还会转发给Judge用于判断是否触发告警。

### **alarm组件**
     alarm模块是处理报警event的，judge产生的报警event写入redis，alarm从redis读取处理

### **graph组件**
     graph是存储绘图数据的组件。graph组件 接收transfer组件推送上来的监控数据，同时处理query组件的查询请求、返回绘图数据。

### **query组件**
     提供统一的绘图数据查询入口。query组件接收查询请求，根据一致性哈希算法去相应的graph实例查询不同metric的数据，然后汇总拿到的数据，最后统一返回给用户。

### **dashboard组件**
     dashboard是面向用户的查询界面。在这里，用户可以看到push到graph中的所有数据，并查看其趋势图。

### **fe组件**
     导航界面

### **nodata组件**
     nodata用于检测监控数据的上报异常。nodata和实时报警judge模块协同工作，过程为: 配置了nodata的采集项超时未上报数据，nodata生成一条默认的模拟数据；用户配置相应的报警策略，收到mock数据就产生报警。采集项上报异常检测，作为judge模块的一个必要补充，能够使judge的实时报警功能更加可靠、完善

### **task组件**
     task是监控系统一个必要的辅助模块。定时任务，实现了如下几个功能：
     index更新。包括图表索引的全量更新 和 垃圾索引清理。
     falcon服务组件的自身状态数据采集。定时任务了采集了transfer、graph、task这三个服务的内部状态数据。
     falcon自检控任务。


# 部署
> 部署使用的二进制包部署，方便快捷，agent使用ansible安装到所有机器上，设置systemctl启动。其他组件包括redis安装到运维机器上。

参照官网的部署
这里略过

# 数据采集
>falcon 数据采集分为以下三种方式

* =基础库= 业务代码加载基础库、服务实例在运行过程中主动采集并push相关数据到agent
* =nginx_lua_module= 嵌在nginx中主动采集业务相关指标 如状态码次数，QPS，服务可用度等
* =自定义数据采集=  自己通过脚本push到agent，查看https://git.chunyu.me/op/op_tools/blob/master/op_tools/falcon.py 这个很灵活 可以采集很多东西

## 数据模型
![](http://or2jd66dq.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-09%20%E4%B8%8B%E5%8D%886.27.47.png)

* metric: 
    * 最核心的字段，代表这个数据采集项具体度量的是什么东西，比如内存的使用量、某个接口的调用次数
* endpoint: 
    * 监控实体，代表metric的主体，比如metric是内存使用量，那么endpoint就表示该metric属于哪台机器，也可以表示某个业务比如medweb，其实endpoint是一个特殊的tag。
* tags: 
    * 这是一组逗号分隔的键值对，用来对metric进行进一步的描述，比如service=falcon,location=beijing
* timestamp: 
    * UNIX时间戳，表示产生该数据的时间
* value: 
    * 整型或者浮点型，代表该metric在指定时间点的取值
* step: 
    * 整型，表示该数据采集项的汇报周期，这对于后续的监控策略配置、图表展示很重要，必须明确指定
* counterType: 
    * 只能是COUNTER或者GAUGE二选一，前者表示该采集项为计数器类型，后者表示其为原值；对于计数器类型，告警判定以及图表展示前，会被先计算为速率


