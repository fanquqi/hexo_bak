---
title: MySQL使用记录
date: 2017-07-13 16:02:23
tags: mysql 
categories: 数据库
---

数据库现在为止我只接触过MySQL，MySQL自从被oracle收购后，就衍生出三大阵营，Drizzle、MariaDB和Percona Server（包括XtraDB引擎），第一个没接触过不太了解，MariaDB与percona又有很多相似点，比如都支持XtraDB（innodb加强版）
看看两者优缺点：

- Percona团队的最终声明是“Percona Server是由Oracle发布的最接近官方MySQL Enterprise发行版的版本”，因此与其他更改了大量基本核心MySQL代码的分支有所区别。
- XtraDB 存储引擎是完全的向下兼容，在 MariaDB 中，XtraDB 存储引擎被标识为”ENGINE=InnoDB”，这个与 InnoDB 是一样的，所以你可以直接用XtraDB 替换掉 InnoDB 而不会产生任何问题。
- MariaDB的目的是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。在存储引擎方面，10.0.9版起使用XtraDB（名称代号为Aria）来代替MySQL的InnoDB。
- Percona Server的一个缺点是他们自己管理代码，不接受外部开发人员的贡献，以这种方式确保他们对产品中所包含功能的控制。

总之，二者目前为止没有拉开什么差距，所以可以自由选择。
阿里使用的percona 但是Google使用的MariaDB。。。

我也是用的percona。下边以它为例说明。

数据库也是个比较大的技术栈，三言两语说不完，工具书都特别厚，我写这篇文章的目的无非就是想给自己一个查看SQL语句的地方，见识比较少，有错误我肯定不是故意的，全文参考这个[技能图谱](http://lib.csdn.net/maquealone/352262/chart/MySQL)

# 安装部署

直接yum就可以，国内有清华源，中科院源，比较快，官网下载简直是灾难。
可以写ansible-playbook，没有难点不多说。


## 配置文件详解

```
[mysqld3306]

# 为PXC定制
binlog_format=ROW
default-storage-engine         = InnoDB
innodb_autoinc_lock_mode=2


# 绑定地址之后只能通过内部网络访问

bind-address=10.******

auto-increment-increment = 2
auto-increment-offset = 1

# 使用 xtrabackup-v2 必须制定datadir, 因为操作可能直接对 datadir进行
datadir=/home/mysql/3306/data

# http://www.percona.com/doc/percona-xtrabackup/2.2/innobackupex/privileges.html#permissions-and-privileges-needed
# 权限的配置
# xtrapbackup在Donor上执行，因此只需localhost的权限



# 约定:
server-id = 1


# wsrep模式不依赖于GTID
# 开启GTID
# enforce_gtid_consistency=1
# gtid_mode=on

# 即便临时作为Slave，也记录自己的binlog
log-slave-updates=1

# master_info_repository=TABLE
# relay_log_info_repository=TABLE

# GENERAL #
user                           = mysql
port                           = 3306
socket                         = /home/mysql/3306/mysql.sock
pid-file                       = /home/mysql/3306/mysql.pid

# MyISAM #
key-buffer-size                = 32M
myisam-recover                 = FORCE,BACKUP


ft-min-word-len = 4
event-scheduler = 0

# SAFETY #
max-allowed-packet             = 16M

skip-name-resolve
max_connections = 2000
max_connect_errors = 30

back-log = 500

character-set-client-handshake = 1
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

#character-set-client-handshake=1
#character-set-client=utf8
#character-set-server=utf8
#collation-server=utf8_general_ci



#key-buffer-size = 256M
table-open-cache = 2048
max-allowed-packet = 2048M
slave-skip-errors = all                       #Skip duplicated key
sort-buffer-size = 4M
join-buffer-size = 8M
thread-cache-size = 50
concurrent-insert = 2

thread-stack = 192K
net-buffer-length = 8K
read-buffer-size = 256K
read-rnd-buffer-size = 16M
bulk-insert-buffer-size = 64M

# 采用thread pool来处理连接
thread_handling=pool-of-threads


sql-mode                       = STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_AUTO_VALUE_ON_ZERO,NO_ENGINE_SUBSTITUTION,ONLY_FULL_GROUP_BY
sysdate-is-now                 = 1
innodb                         = FORCE
innodb-strict-mode             = 1


# BINARY LOGGING #
log-bin                        = /home/mysql/3306/data/mysql-bin
expire-logs-days               = 5
# LOGGING #
# log-output=file 默认就是FILE
log-error                      = /home/mysql/3306/data/mysql-error.log
long-query-time = 0.3
# log-queries-not-using-indexes  = 1
slow-query-log                 = 1
slow-query-log-file            = /home/mysql/3306/data/mysql-slow.log

# 默认为0(MySQL不控制binlog的输出)
# sync-binlog                    = 1

# CACHES AND LIMITS #

tmp-table-size                 = 32M
max-heap-table-size            = 32M

# 频繁修改的表不适合做query-cache, 否则反而影响效率
query-cache-type               = 0
query-cache-size               = 0
# query-cache-limit = 2M
# query-cache-min-res-unit = 512

thread-cache-size              = 100
open-files-limit               = 65535
table-definition-cache         = 1024
table-open-cache               = 4096

# INNODB #
innodb-flush-method            = O_DIRECT
innodb-log-files-in-group      = 2

# innodb-file-per-table = 1设置之后， 下面的配置基本失效
innodb_data_file_path = ibdata1:10M:autoextend
innodb-thread-concurrency = 32
innodb-log-file-size           = 256M
innodb-flush-log-at-trx-commit = 2
innodb-file-per-table          = 1
# 内存: 全部内存*0.7
innodb-buffer-pool-size = 25G

performance-schema = 0
net-read-timeout = 60

# innodb-open-files 在MySQL5.6 auto-sized
# 来自May2
innodb-rollback-on-timeout
innodb-status-file = 1


# http://dev.mysql.com/doc/refman/5.6/en/innodb-performance-multiple_io_threads.html
# http://zhan.renren.com/formysql?tagId=3942&checked=true
# 从MySQL 5.5
# innodb_file_io_threads = 4
innodb-read-io-threads = 16
innodb-write-io-threads = 8

innodb-io-capacity = 2000
# innodb-stats-update-need-lock = 0 # MySQL 5.6中无效
# innodb-stats-auto-update = 0
innodb-old-blocks-pct = 75
# innodb-adaptive-flushing-method = "estimate"
# innodb-adaptive-flushing = 1

# https://www.facebook.com/notes/mysql-at-facebook/repeatable-read-versus-read-committed-for-innodb/244956410932
# READ-COMMITTED 每次QUERY都要求调用: read_view_open_now, 而REPEATABLE-READ每次Transaction中只要求一次
# REPEATABLE-READ 会导致读写不同步
transaction-isolation = READ-COMMITTED

innodb-sync-spin-loops = 100
innodb-spin-wait-delay = 30

innodb-file-format = "Barracuda"
innodb-file-format-max = "Barracuda"
```

# 基本操作
> 日常操作无非就是增删改查。也可分为库层面操作，与表层面操作。

## 库操作

一般库操作很少，没有什么花样
```
# 创建数据库
create database online_database;        
# 查看数据库
show databases;  
# 查看数据库信息    
show create database online_database;
# 修改数据库的编码，可使用上一条语句查看是否修改成功
alter database online_database default character set gbk collate gbk_bin;      
# 删除数据库
drop database online_database;

```

