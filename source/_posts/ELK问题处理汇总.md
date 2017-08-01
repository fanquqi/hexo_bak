---
title: ELK问题处理汇总
date: 2017-06-05 16:46:11
tags: ELK
categories: 运维工具
---

## 1. Field data loading is forbidden on [FIELDNAME]  

> 问题描述： 当我们在kibana画图时，有的nginx_access 日志中的有一部分字段是不能用的，在grafana中也不可以，但是别的索引文件可以，比如elapsed_log,search_log,错误如下图所示

![](http://or2jd66dq.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-05%20%E4%B8%8B%E5%8D%887.06.50.png)
### 问题原因

问题原因就是我们索引的设置中 `"fielddata" : { "format" : "disabled" }` 这个设置存在导致的结果就是

- Field data loading is forbidden on type
- Field data loading is forbidden on path
- Field data loading is forbidden on host

查看索引settings
```
curl 10.215.33.36:9200/logstash-*/_mapping
```
查看确实存在`"mapping":{"fielddata":{"format":"disabled"}`这种设置，

### 解决过程

修改mapping，ES的索引是已建成就固定的，中途修改只能是 建新索引-->数据导入-->旧索引删除。 所以我们可以直接修改template，等到第二天自动更新的时候新的索引设置就会正常了。
具体操作记录在[ES使用](https://fanquqi.github.io/2017/06/05/ElasticSearch%E4%BD%BF%E7%94%A8/)的template更新中。

参考：https://github.com/elastic/elasticsearch/issues/15267


---------



## 2. 修改字段类型
> 数据通过logstash 的grok过滤之后，传输到ES当中，ES是可以自动发现的，自动生成mapping，并不用事先设置mapping，很方便。默认会把所有的字段设置成string,但是有的时候我们需要float等数据类型，怎么更改？？

当前看有两种方式：
1. 直接修改mapping。
2. 在logstash grok的时候指定数据类型。
    + 类似于这种`%{NUMBER:status:float}` ,`NUMBER`是grok支持的数据类型，`status`是自定义字段名称, `float`即为字段类型。但是这个可能不会立即生效。


## 3. mapping 冲突解决
> 问题2中第二种方法修改完字段类型之后，刷新索引，会造成新老数据的类型不一致导致冲突。

参考: https://dev.sobeslavsky.net/kibana-how-to-solve-mapping-conflict/
其实完全不用理会 等到老的索引过期被删掉就好了


## 4. 自动删除索引脚本
> 收集的日志每天量很大有200多G，自己的三个ES节点存储有限，所以留存维持大约三四天的新鲜日志,需要编写个脚本定期删除索引。

```
#!/bin/bash
today=`date +%Y.%m.%d`;
echo "今天是${today}"
# 获得要删除的日期
# 不指定参数时，默认删除3天前的数据（因为是凌晨删除，所以不含当天）
daynum=3
# 当参数个数大于1时，提示参数错误
if [ $# -gt 1 ] ;then
        echo "要么不传参数，要么只传1个参数!"
        exit 101;
fi
# 当参数个数为1时，获取指定的参数
if [ $# == 1 ] ;then
        daynum=$1
fi
esday=`date -d '-'"${daynum}"' day' +%Y.%m.%d`;
echo "${daynum}天前是${esday}"
curl -XDELETE http://localhost:9200/logstash-nginx_access-${esday}
curl -XDELETE http://localhost:9200/elapsed_log-${esday}
curl -XDELETE http://localhost:9200/file_log-${esday}
curl -XDELETE http://localhost:9200/info_log-${esday}
curl -XDELETE http://localhost:9200/search.log-${esday}

echo "${today}执行完成"
# echo curl -XDELETE http://localhost:9200/aaa-*-${esday}
```

内容如上，然后写进cron每天执行就好了。

## 5. logstash 插入json 格式数据
>大数据那边search_log 是json格式的，安装普通的input插入就会有问题。

修改配置文件 input 中的 file 中 加入 codec => json { charset => "UTF-8” }
![](http://or2jd66dq.bkt.clouddn.com/logstash_json.png)

## 6. ES各节点内存分配
- 配置位置：/usr/local/elasticsearch-2.3.2/bin/elasticsearch.in.sh  
- 三个节点 md6：16G     md10：16G     md11:32G
![](http://or2jd66dq.bkt.clouddn.com/ES%E5%86%85%E5%AD%98%E8%B0%83%E6%95%B4.png)

## 7. elasticsearch 节点安全重启
> elasticsearch集群，有时候可能需要修改配置，增删硬件等操作，需要对节点进行升级等操作。但是服务不能停，如果直接kill掉节点，可能导致数据丢失。而且集群会认为该节点挂掉了，就开始转移数据（这个过程相当好资源，经历过两次，直接kill掉某一节点后集群开始relocation，网卡被打满，正常请求很多超时），当重启之后，它又会恢复数据，如果你当前的数据量已经很大了，这是很耗费机器和网络资源的。

- 第一步：先暂停集群的shard自动均衡
```
 curl -XPUT http://192.168.1.2:9200/_cluster/settings -d'

        {

        "transient" : {

        "cluster.routing.allocation.enable" : "none"

            }        

        }'
```
- 第二步：kill要升级的节点
```
ps aux |grep elasticsearch |awk '{print $2}' |xargs kill
```
- 第三步：恢复集群的shard自动均衡
```
curl -XPUT http://192.168.1.2/_cluster/settings -d'

        {

        "transient" : {

        "cluster.routing.allocation.enable" : "all"

        }

        }'
```

## 8. 误删除node2的存储文件导致elasticsearch集群出现问题出现坏节点 集群状态如下图所示 此时kibana上面已经没有数据
> ES某个节点数据被老大误删除
![](http://or2jd66dq.bkt.clouddn.com/ES%E8%8A%82%E7%82%B9%E8%A2%AB%E8%AF%AF%E5%88%A0%E9%99%A4.png)
重启elasticsearch 各个节点之后还是没有数据插入。
`解决办法：`
* 需要删除坏的索引数据 就是今天的数据 具体方法：找到今天的索引elapsed_log-2016.11.03==>点击"动作"==>点击 “删除”  之后Unassigned 节点会消失， 刷新一下 kibana上数据出现。

## 9. Elasticsearch high disk watermark 问题

问题描述：因为我们每个节点都不是单独只提供一个服务，每台机器的配置和硬盘大小都不尽一致，内存啥的可以很好的处理，但是硬盘大小怎么分配？
比如说有一台机器600G硬盘，剩下两台2T硬盘，这种，如果平均分配恐怕是不行了，小磁盘爆了大磁盘还没有存一半。怎么办？

`问题解决` ES在你想到这个问题之前就已经想到了。操作如下

```
curl -XPUT localhost:9200/_cluster/settings -d '{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "80%",
    "cluster.routing.allocation.disk.watermark.high": "50gb",
    "cluster.info.update.interval": "1m"
  }
}'
```

解读下这三个参数。
`cluster.routing.allocation.disk.watermark.low`  Controls the low watermark for disk usage. It defaults to 85%.如果磁盘使用查过85% 就不回新建分片到这个节点上了，保证磁盘不会被撑爆。
`cluster.routing.allocation.disk.watermark.high` Controls the high watermark. It defaults to 90%.默认值百分之90。如果某节点磁盘使用到达90% ES会自动把此节点上的分片转移到别的节点。也可以设置成某个数值类似上文`50gb`
`cluster.info.update.interval` How often Elasticsearch should check on disk usage for each node in the cluster. Defaults to 30s. 多久检查一次磁盘用量，默认值30s.由于我们不可能在1分钟内写几十GB数据所以这个可以设置的稍长一些。

## 10. 时区问题处理

之前看到过有人说ELK的失去问题，类似kibana显示出来时间与真实时间差8小时，我一直没有遇到，直到今天。。。
`问题描述`: 事情的经过是这样的，我给ES扩容,然后修改了template（目的在给集群优化一下，提高下集群的QPS）但是导致之前grok失效了，第二天来了发现grafana中爬虫统计数据就到今天8点整，然后我意识到自己昨天的操作有问题，然后删除今天的数据重新导入。奇怪的事儿发生了。。我明明删除了几天的nginx_access日志的index但是kibana中还是有到今天八点的数据，我仔细一看今天的数据存在了昨天的索引当中。如下图
![](http://or2jd66dq.bkt.clouddn.com/kibanaUTC.png)
然后我特别不解为啥ES会在8点钟新建一个索引？？？
为啥？

其实是logstash在ES建立的索引，它每天0点建立一个新的索引，kibana也接受这种设定, 在查询和展示时根据用户的时区进行处理。
这导致了, 对于东八区, 2017-07-27日, 8点之前, 只有logstash-2017.07.26这个index, 到8点的时候, 创建新的index logstash-2017.07.27, 即, 对于我们这个时区的人来说, 一天的数据存在了两个index里面
**修改方案一**

更改logstash的设定,使用localtime

**修改方案二** 

不做修改，接受

## 11. 重建索引
之前ELK就是给运维自己用，可以随便折腾，索引都是直接删除，或者配置好template等第二天重建索引再看新的数据，但是PM和CTO有时候都看grafana上的数据，运维操作就要求更规范了啊，其实不是给他们看，我们自己就该严格要求自己，下边记录下重建索引的的过程。
