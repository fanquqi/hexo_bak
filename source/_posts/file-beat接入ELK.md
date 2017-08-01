---
title: file_beat接入ELK
date: 2017-06-06 15:56:13
tags: ELK, filebeat, kafka
categories: 运维工具
---

> 公司之前的日志收集使用的kids(知乎开源的日志收集工具，不知道为啥用这个，难道是因为跟我们你是邻居？)。然后统一收留存到一台机器，做中转留几天的方便大家查看，后续直接存到Hadoop中。后来搭建ELK就是直接在这个机器上起的logstash，说白了只是把这几天的日志做了可视化。但是现在我们有了另一种需求，需要收集各个机器的系统日志，加上之前做了bash_history 也需要收集。但是我并不是很想用kids,它改配置重启都比较麻烦。也不太想用整个的logstash，太大了，正好filebeat可以满足我们的需求，很轻。
> 但是还是有一个想法，之前的ELK 有的时候会出现数据延时很大的情况，我分析是ES集群的压力过大，所以我想在ES之前加一个缓冲，加一个kafka队列
> 这正好是一个实验的机会。


之前架构：
![](http://or2jd66dq.bkt.clouddn.com/ELK%E6%97%A9%E6%9C%9F%E6%9E%B6%E6%9E%84.png)

架构调整如下：

![](http://or2jd66dq.bkt.clouddn.com/ELK-kafka.png)

参考:https://www.ibm.com/developerworks/cn/opensource/os-cn-elk-filebeat/index.html
## 版本介绍
- logstash 2.3.2
- Elasticsearch 2.3.2
- kafka 0.10.0.2
- filebeat 5.0.0
- kibana 4.3.1
由于ELKstack 各个组件之前有很强的版本依赖，所以安装之前最好确认下。据说filebeat5.0 logstash也得使用5.0,ES 也得使用5.0 但是我们使filebeat 和logstash之间加了一个kafka队列，就给这两个组件直接解耦了。filebeat5.0之后才能直接output到kafka。logstash2.3可以直接使用kafka 作为input。正好。。。


# kafka安装

> 最新版本0.10.0.2
> java 1.8

部署参考: https://www.mtyun.com/library/32/how-to-install-kafka-on-centos7/
kafka配置参考: http://www.cnblogs.com/davidwang456/p/4195873.html

`Broker`
Kafka集群包含一个或多个服务器，这种服务器被称为broker

`Topic`
每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。（物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处）

`Partition`
Parition是物理上的概念，每个Topic包含一个或多个Partition.
很像ES的分片概念，合理设置可以提高效率

`Producer`
负责发布消息到Kafka broker

`Consumer`
消息消费者，向Kafka broker读取消息的客户端。

`Consumer Group`
每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。


```
wget http://mirror.bit.edu.cn/apache/kafka/0.10.2.1/kafka_2.10-0.10.2.1.tgz
tar -zxvf kafka_2.10-0.10.2.1.tgz

# 启动zookeeper 
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
# 启动kafka
bin/kafka-server-start.sh config/server.properties &
# 功能验证
## 新建一个topic 
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
## 查看topic 
bin/kafka-topics.sh --list --zookeeper localhost:2181
## 手动产生消息
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
Hello Kafka！
nihao shijie
## 消费消息
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
Hello world!
nihao shijie
```

## 单机多实例配置
> 由于kafka 使用的是机器的文件系统存储，所以本机多实例的集群很好。尽量减少网络通讯。

```
cp config/server.properties config/server_1.properties
# 修改配置文件 
vim config/server_1.properties
    broker.id=1
    port=9093
    log.dirs=/tmp/kafka_1-logs
# 启动
bin/kafka-server-start.sh config/server_1.properties &

```

## 机器调优
File descriptors

kafka会使用大量文件和网络socket，所以，我们需要把file descriptors的默认配置改为100000。修改方法如下(之前机器初始化的时候做过响应更改，可以忽略)
```
#vi /etc/sysctl.conf

fs.file-max = 32000

#vi /etc/security/limits.conf

yourusersoftnofile10000

youruserhardnofile30000
```

    
# filebeat5 安装

> 版本5.0.0

```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.0.0-x86_64.rpm
rpm -ivh filebeat-5.0.0-x86_64.rpm
```
配置

[官网配置参考](https://www.elastic.co/guide/en/beats/filebeat/5.0/configuration-filebeat-options.html)

```
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/bash_history.log*
  fields:
    log_source: bash_history
  document_type: bash_history
  scan_frequency: 1s
  ignore_older: 30m

- input_type: log
  paths:
    - /var/log/messages*
  fields:
    log_source: message
  document_type: message
  scan_frequency: 1s
  ignore_older: 30m
- input_type: log
  paths:
    - /var/log/cron
  fields:
    log_source: cron
  document_type: cron
  scan_frequency: 1s
  ignore_older: 30m
- input_type: log
  paths:
    - /var/log/secure
  fields:
    log_source: secure
  document_type: secure
  scan_frequency: 1s
  ignore_older: 30m
output.kafka:
  hosts: ["localhost:9092", "localhost:9093"]
  topic: '%{[type]}'
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
```

# logstash 配置

> logstash 是把数据从kafka中拿过来，放到ES中配置如下

```
input {
  kafka {
    zk_connect => "localhost:2181"
    topic_id => "sys_log"
  }
}

output {

    elasticsearch {
        hosts => ["10.215.33.36:9200"]
        manage_template => true
        index => "syslog-%{+YYYY.MM.dd}"
    }
}
```



