---
title: ELK搭建
date: 2017-06-05 16:42:39
tags: ELK 
categories: 运维工具

---

> 我来之前公司是有一套ELK服务的，但是不知道为何，没有推广起来，之前的运维都离职，原因不得而知，但是公司每天日志量还是挺大的分析起来十分费力，每天上完线之后有问题就得去机器上tail -f 各种grep  AWK sort 定位问题有点慢。另外考虑到所有的nginx_access,elapsed，filelog，searchlog等都被收集到Hadoop的Hbase中，所以我想从中间截一下，用logstash把日志收集到ES集群，然后通过kibana展示出来

# 软件版本

版本：（ELK版本相互很依赖，所以一定要注意看`README.txt` 不要做无用功）

* Logstash：2.3.2
* elasticsearch: 2.3.2
* kibana: kibana-4.3.1-linux-x64

[升级教程](http://jerrymin.blog.51cto.com/3002256/1927481)

## 部署过程
> 刚开始的架构比较简单，只是logstash + ES集群(三个节点) + kibana
> 
## Elasticsearch部署

### 解压到/usr/local/
```
unzip elasticsearch-2.3.2.zip -d /usr/local/
```


**插件安装**

1. Elasticsearch插件安装方式（推荐）
 

**head 插件安装**
ElasticSearch-Head 是一个与Elastic集群（Cluster）相交互的Web前台。

ES-Head的主要作用

它展现ES集群的拓扑结构，并且可以通过它来进行索引（Index）和节点（Node）级别的操作
它提供一组针对集群的查询API，并将结果以json和表格形式返回
它提供一些快捷菜单，用以展现集群的各种状态

在Elasticsearch目录下
```
./bin/plugin install mobz/elasticsearch-head

```
![](http://or2jd66dq.bkt.clouddn.com/elastic_head.png)
**kopf 插件安装**
Kopf是一个ElasticSearch的管理工具，它也提供了对ES集群操作的API。


```
cd /usr/local/ELK/elasticsearch-2.3.2 && ./plugin install lmenezes/elasticsearch-kopf
启动
open http://localhost:9200/_plugin/kopf

```

![](http://or2jd66dq.bkt.clouddn.com/elastic_kopf.png)

插件访问方式
http://localhost:9200/_plugin/head 
http://localhost:9200/_plugin/koph


2. 下载安装方式

从https://github.com/mobz/elasticsearch-head下载ZIP包。
在 elasticsearch  目录下创建目录plugins/head/ 并且将刚刚解压的elasticsearch-head-master目录下所有内容COPY到当前创建的plugins/head/目录下即可。

 
### 配置文件修改   

node1
```
17: cluster.name: cy_es_cluster           #集群名字（各节点一致）
 23: node.name: node-1                    #节点名称
 43: bootstrap.mlockall: true             #锁住内存
 54: network.host: 10.***.1               #本机内网地址
 58: http.port: 9200
 60: http.cors.enrue
 61: http.cors.allow-origin: "/.*/"
 71: discovery.zen.ping.unicast.hosts: ["10.***.1:9300", “10.***.2:9300”, “10.***.3:9300"]
 75: discovery.zen.minimum_master_nodes: 2
```

node2

```
grep -n '^ [a-Z]' /usr/local/elasticsearch-2.3.2/config/elasticsearch.yml
 17: cluster.name: cy_es_cluster           #集群名字（各节点一致）
 23: node.name: node-2                     #节点名称
 33: path.data: /data2/ES                 
 37: path.logs: /data2/ES/logs          
 43: bootstrap.mlockall: true              #锁住内存
 54: network.host: 10.***.2                #本机内网地址
 58: http.port: 9200                       #端口
 60: http.cors.enabled: true               #kibana3 需要打开 kibana4忽略
 61: http.cors.allow-origin: "/.*/“        #kibana3 需要打开 kibana4忽略
 73: discovery.zen.ping.unicast.hosts: ["10.***.1:9300", “10.***.2:9300”, “10.***.3:9300"]  #集群discovery      
 77: discovery.zen.minimum_master_nodes: 2   #最小集群节点数目
```
       
新加node3在 ,配置一致
    * 启动  nohup /usr/local/elasticsearch-2.3.2/bin/elasticsearch -Des.insecure.allow.root=true &
    * 启动 systemctl start kibana.service
    * 查看各节点启动情况 http://10.***.01:9200/_plugin/head/



## logstash部署
 
```   
tar -zxvf  logstash-all-plugins-2.3.2.tar.gz -C /usr/local/
vim /usr/local/logstash-2.3.2/conf/main.conf 修改配置文件 包括input filter output 
 配置文件详解
 input {
         file {
         path => "/home/chunyu/backup/nginx_log/access.log-*"   日志文件可以模糊匹配
         type => “nginx_access"                                                       类型 在后面output匹配用
         ignore_older => 14400                                                          读取4小时之内文件
         exclude => "*.lzma"                                                               不包括.lzma 结尾的文件
          }
 }
 注意在这遇到过匹配日志日期结尾的情况，例如test.2016-07-03-access.log 匹配为*.%{yyyy}-%{MM}-%{dd}-access.log反复不生效，遂采取ignore_older, 没有在官方找到匹配日期的方式。

 filter {
 if [type] == "nginx_access" {
      grok {        
           patterns_dir => "/usr/local/logstash-2.3.2/patterns/nginx"  指定grok匹配文件
           match => {
                     "message" => "%{NGINXACCESS}"
           }
           }
       geoip {
     source => "clientip"
   }
         }
        }
 grok 正则表达式需要自己编写 对照nginx log format  和nginx具体日志在grok debugger中写完 粘贴到/usr/local/logstash/patterns/nginx中即可


output {
    if [type] == "nginx_access" {
     elasticsearch {
              hosts => ["10.***.01:9200"]
              index => "logstash-nginx_access-%{+YYYY.MM.dd}"
             }
         }
         }
```
output就是把logstash数据输出给elastic search 注意这个index 便是 kibana中setting要输入的

### grok表达式
> 注意，使用过程中有的情况会出现grok失效的情况，类似状态码为500 的时候或者某种情况，你所要过滤的字段在日志中是空的，可以使用“|”等提高容错性。
以nginx_access.log实例
(注意按照自己的日志format灵活变动)
[grokdebug地址](http://grokdebug.herokuapp.com/)

nginx_access.log 格式

```
log_format main '$remote_addr - - [$time_local] "$request" $status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" [$request_time, $upstream_response_time] $host ($remote_port) "sid=$cookie_sessionid"';
```


对应grok表达式
```
NGINXACCESS %{IPORHOST:clientip} - - \[%{HTTPDATE:time_local}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}\" %{NUMBER:status:float} %{NUMBER:body_bytes_sent:float}\ (?:\"(?:%{URI:referrer}|-)\"|%{QS:referrer}) %{QS:agent} \[%{NUMBER:request_time:float}\, (%{NUMBER:upstream_response_time:float}|-)] %{IPORHOST:host} \(%{NUMBER:remote_port}\) %{QS}
```

## kibana 部署 

tar -zxvf kibana-4.1.1-linux-x64.tar.gz -C /usr/local/
grep -n "^ [a-Z]" /usr/local/ELK/kibana-4.3.1-linux-x64/config/kibana.yml
          12: elasticsearch.url: "http://10.***.1:9200” 只需要输入elastic search地址 端口 即可

启动 nohup /usr/local/ELK/kibana-4.3.1-linux-x64/bin/kibana &
登录 10.215.33.36:5601通过—>settings设置index —>查看discovery —>制作visualize —>设置dashboard 达到日志可视化

![图表显示](http://or2jd66dq.bkt.clouddn.com/kibana%E5%9B%BE%E8%A1%A8%E6%98%BE%E7%A4%BA.png)

## 问题处理

1. **地图配置**
kibana地图配置：
默认的地图不能显示，所以我们换成了高德地图
先下载GeoIP库
```
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
gunzip GeoLiteCity.dat.gz
```
vim conf/main.conf 添加下图内容

![](http://or2jd66dq.bkt.clouddn.com/logs_map_logstash_conf.png)


重启logstash

kibana修改
```
vim ./src/ui/public/vislib/visualizations/_map.js +11 
注释  url: 'https://otile{s}-s.mqcdn.com/tiles/1.0.0/map/{z}/{x}/{y}.jpeg',
添加  url: 'http://webrd0{s}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}',   （style=6为卫星地图7为普通地图）
vim ./src/ui/autoload.js +35
末尾 添加  'ui/vislib'
重启kibana
```
之后打开logs.chunyu.me setting重置 但是还是没有地图显示，界面为白板，报错 
Google一下需要改elastic search模板
更改方法：打开 logstash  main.conf配置文件 需要把output 的index 改为logstash开头使用elasticsearch默认模板即可

![](http://or2jd66dq.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-15%20%E4%B8%8B%E5%8D%886.37.57.png)

重启logstash
效果图

![](http://or2jd66dq.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-15%20%E4%B8%8B%E5%8D%886.39.17.png)

## lucene语法
> ELK中，kibana或者grafana用来展示数据，Elasticsearch是搜索引擎也是存储中心，它是构建在Lucene之上的，所以过滤器语法和Lucene相同。


版本：（ELK版本相互很依赖，所以一定要注意看README.txt 不要做无用功）

* Logstash：2.3.2
* elasticsearch: 2.3.2
* kibana: kibana-4.3.1-linux-x64

[参考链接1](https://kibana.logstash.es/content/elasticsearch/api/search.html)
[参考链接2](https://segmentfault.com/a/1190000002972420)

`全文搜索`
500
显示带有500 的内容

`短语搜索`
使用双引号包起来
"like Gecko"

`字段`

也可以按页面左侧显示的字段搜索

限定字段全文搜索:field:value

`精确搜索`：关键字加上双引号 filed:"value"

status:404 搜索http状态码为404的文档
字段本身是否存在

`_exists_`:http：返回结果中需要有http字段

`_missing_`:http：不能含有http字段
通配符

? 匹配单个字符

* 匹配0到多个字符
kiba?a, el*search

? * 不能用作第一个字符，例如：?text *text

正则

es支持部分正则功能

mesg:/mes{2}ages?/
模糊搜索

~:在一个单词后面加上~启用模糊搜索

first~ 也能匹配到 frist

还可以指定需要多少相似度

cromm~0.3 会匹配到 from 和 chrome

数值范围0.0 ~ 1.0，默认0.5，越大越接近搜索的原始值
近似搜索

在短语后面加上~

"select where"~3 表示 select 和 where 中间隔着3个单词以内
范围搜索

数值和时间类型的字段可以对某一范围进行查询

length:[100 TO 200]

date:{"now-6h" TO "now"}

[ ] 表示端点数值包含在范围内，{ } 表示端点数值不包含在范围内
逻辑操作

AND

OR

NOT
+：搜索结果中必须包含此项

-：不能含有此项
+android -OPPO 包含android 不包含OPPO 
+apache -jakarta test：结果中必须存在apache，不能有jakarta，test可有可无 

