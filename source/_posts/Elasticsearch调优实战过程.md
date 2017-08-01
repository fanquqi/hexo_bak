---
title: ElasticSearch调优实战过程
date: 2017-06-06 13:57:27
tags: ElasticSearch
---

目前的集群有三个节点，配置如下
![](http://or2jd66dq.bkt.clouddn.com/kopfESnodestatus.png)
可以看到三个节点都是master 都是data节点，分工并不明确分配的内存和磁盘不尽相同。现在的集群还是处于一个半原始的状态，没有做太多的优化，正好最近要扩容集群，我顺便优化一下，打算新开一台机器，把新节点用docker起，慢慢换，一步一步来。


关于集群的设置可以直接通过API传上去，也可以通过kopf插件更改(建议新手)

## 优化一 合理规划节点分片 

ES使得创建大量索引和超大量分片非常地容易，但更重要的是理解每个索引和分片都是一笔开销。所以什么都得辩证得看，能结合业务做到合适是最重要的。索引分片就是把索引数据切分成多个小的索引块，这些小的索引块能够分发到同一个集群中的不同节点。在检索时，检索的结果是该索引每个分片上检索结果的总和(尽管在某些场景中“总和”并不成立：单个分片有可能存储了所有的目标数据)。默认情况下，ElasticSearch会为每个索引创建5个主分片，就算是单结点集群亦是如此。像这样的冗余就称为过度分配：这种场景下，这种分配方式似乎是多此一举，而且只会在索引数据(把文档分发到多个分片上)和检索(必须从多个分片上查询数据然后全并结果)的时候增加复杂度，乐享其成的是，这些复杂的事情是自动处理的，但是为什么ElasticSearch要这样做呢？

## 优化二 限制内存使用

> [参照官网](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_limiting_memory_usage.html)
> 一旦分析字符串被加载到 fielddata ，他们会一直在那里，直到被驱逐（或者节点崩溃）。由于这个原因，留意内存的使用情况，了解它是如何以及何时加载的，怎样限制对集群的影响是很重要的。


## 优化三 缓存 
[参考地址](https://wizardforcel.gitbooks.io/mastering-elasticsearch/content/chapter-5/54_README.html)
清空缓存
curl -XPOST 'localhost:9200/_cache/clear'

未完待续。。。
