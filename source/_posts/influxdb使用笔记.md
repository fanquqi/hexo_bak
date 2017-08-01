---
title: influxdb使用笔记
date: 2017-07-06 20:00:36
tags: influxdb
categories: influxdb
---
> 之前从ucloudAPI上取下来的数据需要存到influxdb中，在grafana中进行展示。grafana是我自行部署推广使用的，influxdb是别的同事之前就开始用了。我这次正好用上，所以仔细看了下。虽然人家都把API封装好了，只是拿过来就用的事儿。但是我只是个爱学习的孩子。。。

## 介绍
> 关于时序数据库，除了常用的ElasticSearch之外，InfluxDB也是一个选择。

InfluxDB 使用 go 语言编写。个人认为几个外在的优点在于：

- 无特殊依赖，几乎开箱即用（如ES需要Java）；
- 自带HTTP管理界面，免插件配置（如ES的kopf或者head）；
- 自带数据过期功能；
- 类SQL查询语句（再提ES，查询使用自己的DSL，虽然也可以通过sql插件来使用类SQL进行查询）；
- 自带权限管理，精细到“表”级别；

## 关键词解读
参考[官方文档](https://docs.influxdata.com/influxdb/v1.0/concepts/glossary)

- `time`  这个概念首先说明，influxdb本身就是一个时序数据库类似Elasticsearch，所以用来做流处理是很好的，一次插入，多次读写，少改动。 每次插入一条数据都必须要求一个时间戳，自己不定义他就自动生成。
- `database`  就是数据库
- `measurement` 相当于mysql中的table
- `field` 类似于MySQL的字段，没有索引的列，是influxDB数据必须的组成部分。
- `tags` 相当于MySQL带索引的字段，不必须
- `retention policy (RP)` 描述数据存储多久，以及规定几个分片
- `point` 同一时间戳产生的数据集合
- `sereis`  measurement, tag set, and retention policy 都相同的数据集合


## CLI命令
> CLI[官方文档](https://docs.influxdata.com/influxdb/v1.2/introduction/getting_started/)
> influx的CLI命令与mysql还是有很多相似之处的。不过用influxdb我们更多的是用他的API很少用到CLI，只是了解下，自己调试代码的时候可以验证一下自己的数据到底写没写进来。


#### 建库

```
CREATE DATABASE mydb

验证：
SHOW DATABASES

使用：
USE mydb
```

现在，建好库可以插入数据了。

**数据类型**
只支持以下几种
`float`,`integer`,`string`,`Boolean`,`Timestamp`

#### 建表并插入数据
这条命令表示: 新建一个表名为`cpu`的表，设置`host`为`serverA`,`region`为`us_west`,`value`为0.64,其中 **host,region** 属性为`tags`, **value** 属性为`field`
```
INSERT cpu,host=serverA,region=us_west value=0.64
```

#### 查询数据
```
SELECT "host", "region", "value" FROM "cpu" WHERE "value">0.9
```

## HTTP API使用
[官方文档查看](https://docs.influxdata.com/influxdb/v1.0/guides/writing_data/)

建库
```
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"
```

单条数据写入

```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
```

多条数据写入
```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server02 value=0.67
cpu_load_short,host=server02,region=us-west value=0.55 1422568543702900257
cpu_load_short,direction=in,host=server01,region=us-west value=2.0 1422568543702900257'
```


