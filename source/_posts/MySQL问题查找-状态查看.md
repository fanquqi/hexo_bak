---
title: 'MySQL问题查找,状态查看'
date: 2017-08-02 16:35:18
tags: MySQL
categories: 数据库
---

导火索: 今天后端上线，上线脚本执行到数据库更改的时候卡住了(公司开发的devops,web操作)，登录机器查看ansible进程是在的，证明SQL 执行出错了。之前我很少接触mysql，全是部署增删改查，多表查询都少用，所以这次是一边Google一边查找原因。

首先，这次数据库修改一共有四处，前三个都已经执行完成，就是最后一个加字段的没有成功。

**排查过程**

1. 查看表大小，确认不是因为表大导致执行时间过长。
    其实这个再上线操作的时候又应该有个大概的认识，比如这个表功能大概是多少行，属于哪个工程等等，如果对业务熟悉，应该很快判断的出。
    `相关命令`
    ```
    // 直接查找，可能较慢
    select count(id) from table_name; 
    // 查找information_schema中的信息
    select * from information_schema.TABLES 
        where information_schema.TABLES.TABLE_SCHEMA='databasename'
        and information_schema.TABLES.TABLE_NAME='tablename'\G
    ```

2. 如果不是表太大，可以看下正在进行的事务
    了解事务的工作模式很重要，首先要知道自己的MySQL隔离级别，及每种隔离级别的特点。参考[美团点评团队的分享](https://tech.meituan.com/innodb-lock.html) 我们是使用的 已提交读（Read committed）
    ```
    // 查看当前隔离级别
    SELECT @@session.tx_isolation;
    // 查看当前执行的事务
    SELECT * FROM information_schema.INNODB_TRX;
    ```
    当时主库加上字段之后，从库一直加不上字段就是因为有另一个事务一直没有提交，导致表一直是只读状态，阻塞了。


select * from information_schema.TABLES 
        where information_schema.TABLES.TABLE_SCHEMA='online_medical'
        and information_schema.TABLES.TABLE_NAME='api_familydoctorservice'\G
