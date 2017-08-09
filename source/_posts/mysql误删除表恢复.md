---
title: mysql误删除表恢复
date: 2017-08-04 17:12:09
tags: MySQL
categories: 数据库
---

# 场景描述
 今天有个开发哥哥来找我。说有个医生信息表被删除了（测试服）但是现在版本正在测试，账号已经登不上了，bug群里已经有人说话了，CTO已经看到了，情况已经很紧急了。但是我之前也没有遇到过这类事情，而且测试服mysql是不备份的，一时不知道要怎么办。

# 解决

**但是** 事情还是要解决也总是有解决办法。下边按照先后顺序记录了下我的操作方案。

## 方案一

> 使用bin-log，我们测试服mysql是有记录mysql-binlog的，（本来也是没有开的，在六月份的时候测试服总是内存不够用，我把数据库从本地迁移独立了出来开启了bin-log）

**bin-log恢复方案介绍**

先查看这次今天mysql操作记录，发现没有什么操作，毕竟周五， 所以计划直接恢复到删表之前（就是昨天，中间操作的一段时间有几个今天修改的记录不见得情况），然后再从删表后的时间节点到现在恢复，等于只跳过了那条删表操作，别的操作都重新来一遍。

### 操作流程

首先查看mysql-binlog 找到误操作drop table的时间节点 这里没有图文。。。
    
    // 查找对应的bin-log文件
    mysqlbinlog mysql-bin.000002 | grep -A4 "table_name" 

然后恢复对应时间节点恢复（其实mysqlbinlog也可以对应postion节点恢复）
    mysqlbinlog -d test_medical --stop-datetime='2017-08-03 17:57:05' /home/mysql/3306/data/mysql-bin.000002 | mysql -uroot -p -P 3306 -S /home/mysql/3306/mysql.sock
    // -d 数据库名  --stop-datetime 结束时间点，--start-datetime 开始时间节点 二者可以一起写就是中间时间短的操作 

    
可想而知我这样操作的结果，肯定是一堆冲突，因为当前数据库已经是操作过一遍的结果了，再把执行过得命令拿过来执行下，肯定是有错误。
所以我们方案一肯定是不可取的。

![](http://or2jd66dq.bkt.clouddn.com/binlog-error.png)


## 方案二

> 我之前是有迁移过这个MySQL的，我赶紧看了下迁移的数据压缩包还在时间是2017-6-19,顿时感觉欣喜若狂

我的设想是这样的，因为时间久远肯定不能再这个mysql上恢复到一个多月之前，我想把数据包下到本地电脑mysql，恢复数据，把这个表的dump下来，传到测试服mysql。然后数据包压缩完之后有3G多 下载限速1M大约一个小时。
太久了。
我索性在测试数据库服务器上 用ansible由装了一个mysql实例端口3309用时两分钟不到。

恢复数据 [参考文档](https://fanquqi.github.io/2017/06/16/%E5%87%A0%E7%A7%8Dmysql%E8%BF%81%E7%A7%BB/)

然后用mysqlbinlog恢复这一个多月来的数据。

    mysqlbinlog -d test_medical --stop-datetime='2017-08-03 17:57:05' /home/mysql/3306/data/mysql-bin.000001 | mysql -uroot -p -P3309 -S /home/mysql/3309/mysql.sock
    
    mysqlbinlog -d test_medical --stop-datetime='2017-08-03 17:57:05' /home/mysql/3306/data/mysql-bin.000002 | mysql -uroot -p -P3309 -S /home/mysql/3309/mysql.sock

备份表

    mysqldump -uroot -p test_medical table_name > table_name.sql

然后进到mysql source一下  结束。。


# 总结

binlog是结合备份来用的，其实我们这次恢复也算是 `备份—恢复`流程

所以

数据库备份很重要，

数据库备份很重要，

数据库备份很重要。 


