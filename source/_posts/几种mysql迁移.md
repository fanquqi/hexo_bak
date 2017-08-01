---
title: 几种mysql迁移
date: 2017-06-16 12:12:23
tags: MySQL
categories: 数据库
---

> 运维在日常免不了迁移服务，而迁移服务又免不了迁移数据库。 通常情况下数据库还比较重要，所以怎么样平滑的迁移数据库就显得很重要。
> 我在前段时间迁移了青云几台机器的服务，其中涉及到一些MySQL数据库的操作，做了些记录。

-------

## mysqldump

> 这是比较常见，一般数据库数据量很小的时候，首要我们就会考虑这个。
`优点`:在于能够与正在运行的 MySQL 自动协同工作，支持备份 InnoDB 以及 MyISAM 表；锁表既是优点又是缺点，看怎么看待；
`缺点`:在于数据量大，备份速度慢；

这次服务迁移 中涉及到我们的OA系统，之前是部署在一个windows-server上的。
### 需求
1. windows 上mysql迁移到 centos7  
2. 之前单实例换为双主(haproxy代理)

### 迁移具体过程
在centos7机器 
直接远程dump
```
 mysqldump -h117.******* -uoa -p oa --flush-logs --single-transaction | mysql -h localhost -uoa -p -P3314 -S /home/mysql/3314/mysql.sock v50
```

`--flush-logs`
在开始导出前刷新服务器的日志文件。注意，如果你一次性导出很多数据库（使用 -databases=或--all-databases选项），导出每个库时都会触发日志刷新。例外是当使用了--lock-all-tables或--master-data时：日志只会被刷新一次，那个时候所有表都会被锁住。所以如果你希望你的导出和日志刷新发生在同一个确定的时刻，你需要使用--lock-all-tables，或者--master-data配合--flush-logs。

`--single-transaction`
InnoDB 表在备份时，通常启用选项 --single-transaction 来保证备份的一致性，实际上它的工作原理是设定本次会话的隔离级别为：REPEATABLE READ，以确保本次会话(dump)时，不会看到其他会话已经提交了的数据。

之后查看windows机器的bin-log日志，查看在备份时间节点新的bin-log文件
![](http://or2jd66dq.bkt.clouddn.com/windows_binlog.png)

之后建立两个mysql主从(期间我centos7机器的mysql双主已经使用ansible搭建成功在此省略)

windows——MySQL 授权
```
设置chunyu账号负责数据同步
GRANT REPLICATION SLAVE,RELOAD,SUPER ON *.* TO 'chunyu'@'%' IDENTIFIED BY '123';
FLUSH PRIVILEGES ;
```

centos7_1 停掉双主中自己slave角色，重新设置master
```
STOP SLAVE;
RESET SLAVE;
change master to master_host='117.****',master_port=3306,master_user='chunyu',master_password='123',master_log_file='mysql-bin.000008',master_log_pos=120;
start slave;
show slave status;
```

因为centos是双主，所以需要先断掉这台机的slave，另一台centos不用操作，这样双主变成了主从，在设置windows——MySQL为主此时关系应该是:
win_mysql(master) centos7_1(slave)
centos7_1(master) centos7_2(slave)

之后服务迁移完之后，可以再设置回双主此处省略。



## percona-xtrabackup

> 这台是redmine的数据库数据比较多，部署在ubuntu上。数据量大可以使用xtrabackup 热备份。

`优点`: 可靠高效的备份DB;备份过程中不中断事务处理，热备份;快速进行恢复等。
`缺点`:

### 需求
1. ubuntu上mysql迁移到centos7 mysql
2. 之前单实例换为双主(haproxy代理)

### 操作过程

> 也是先备份传输，建立主从，恢复双主。


ubuntu上 备份
```
apt-get install percona-xtrabackup
innobackupex --user=root --password=123 ./
tar -czvf mysql_back.tar.gz 2017-04-25_10-58-40
```

centos7 传输 解压 放到相应目录
```
rsync -auvzP --bwlimit=5000 root@117.****:/mnt/sdc/mysql_back.tar.gz ./
cd /home/mysql/ && mv data/data_back
tar -zxvf mysql_back.tar.gz 
mv 2017-04-25_10-58-40/ data
chown -R mysql:mysql data
```

postion点查看:
```
[root@test_biz 2017-06-16_15-39-01]# cat xtrabackup_binlog_info
mysql-bin.000054    57952335
```


之后主从设置 迁移完服务之后恢复双主。





