---
title: open-falcon升级
date: 2017-08-15 11:08:50
tags: openfalcon
categories: 监控
---

> 这次迁移本来想使用docker，看了下官方给的镜像2G多，差点吓死我，另外还有mysql，redis融合在里面。我们的mysql不在机器本地，也没有打算迁移。为了防止上次ucloud  B机房整体服务断绝联系之后一个报警都没有尴尬发生，我准备把这台机器放到其他机房，万一下次ucloud B机房挂掉最起码监控是有的，但是如果它全线挂。。。。呸，上次就是我这乌鸦嘴。。。


&emsp;&emsp;这次迁移过程具体分为

- falcon—plus 部署
- 新系统测试
- 各机器falcon-agent升级
- 域名更改，grafana更改

# falcon-plus部署

&emsp;&emsp;参考[官网](https://book.open-falcon.org/zh_0_2/quick_install/upgrade.html)
&emsp;&emsp;首先数据库有更改，增加了一个[alarm库](https://github.com/open-falcon/falcon-plus/blob/master/scripts/mysql/db_schema/5_alarms-db-schema.sql) 这个是用来存储报警历史的，在dashboard中有展示，需要我们down下来执行下这个SQL。然后`sender`模块和`alarm`模块合二为一了，也就是说报警直接由`alarm`模块发送了，这点我刚看到的时候有一点纠结，因为v0.1版本我们没有使用`sender`模块,而是直接使用的一个脚本实时读redis，然后根据级别判断，发送邮件，微信，短信，钉钉，这不知道是不是意味着我必须修改`alarm`源码了。我们先厚着头皮继续部署，遇到什么问题解决什么问题。

机器初始环境部署我在这里忽略了，包括golang等一些环境的安装。

**执行SQL**
    wget https://github.com/open-falcon/falcon-plus/blob/master/scripts/mysql/db_schema/5_alarms-db-schema.sql > mysql -h127.0.0.1 -ufalcon -p

**二进制安装**

安装包下载[地址](https://github.com/open-falcon/falcon-plus/releases/download/v0.2.0/open-falcon-v0.2.0.tar.gz)

```
cd $falcon_dir
wget https://github.com/open-falcon/falcon-plus/releases/download/v0.2.0/open-falcon-v0.2.0.tar.gz
tar -zxvf open-falcon-v0.2.0.tar.gz
```

然后对应修改每个模块的配置文件
这个我就不记录了

**启动|停止|检查** 
    ./open-falcon start|stop|check 

**各节点启动方式**

1. 进到相应模块bin目录，执行二进制文件 例：./falcon-agent start|stop|restart|tail 
2. 在$faocon_dir ./open-falcon start|stop|reload|monitor


## dashboard部署
&emsp;&emsp;参考[官网](https://github.com/open-falcon/dashboard)

操作按照[官网文档](https://book.open-falcon.org/zh_0_2/quick_install/frontend.html)依次进行

这里有个小坑
配置openLDAP登录的时候，配置
配置也较之前0.1版本有所不同）

0.1版本可以用reader用户+密码去验证用户名密码

        "ldap": {
        "enabled": true,
        "addr": "ldap.example.com:389",
        "baseDN": "dc=example,dc=com",
        "bindDN": "cn=reader,dc=example,dc=com",
        "bindPasswd": "123456",
        "userField": "uid",
        "attributes": ["uid","givenName","sn","mail"]

0.2版本验证方式直接是向openLDAP的389端口验证该用户名的密码正确性，不用配置reader用户和密码

    LDAP_ENABLED = os.environ.get("LDAP_ENABLED",True)
    LDAP_SERVER = os.environ.get("LDAP_SERVER","ldap.example.com:389")
    LDAP_BASE_DN = os.environ.get("LDAP_BASE_DN","uid=%s,OU=People,DC=example,DC=com")
    LDAP_BINDDN_FMT = os.environ.get("LDAP_BINDDN_FMT","uid=%s,OU=People,DC=example,DC=com")
    LDAP_SEARCH_FMT = os.environ.get("LDAP_SEARCH_FMT","(uid=%s)")
    LDAP_ATTRS = ["uid","givenName","sn","mail", "cn"]

具体LDAP概念比如cn sn [参考](https://segmentfault.com/a/1190000002607140)

配置完成。但是。。之前的用户不能登录成功。。。
看源码看到这句
![](http://or2jd66dq.bkt.clouddn.com/open-falcon-dashboard-bug.png)
WTF，so，只能删掉用户数据，重建，但是。。。

&emso;&emsp;我把用户都删了重建，这个导致了后来一个bug，因为报警都是按组发送的，所以用户删完之后，组别在dashboard中没有显示了。。新建会显示 **组已经存在** 我去，原来数据库中还有一个action表，里面还有原来分组的消息，我只能新建另一个分组，到portal中（v0.1版本我没有下掉，监控不能断）把报警接收组改成新加的这个组。。

然后就OK了。
但是。上文说过，alarm与sender合并了，所以报警怎么发？

我去看了下alarm的配置文件
发现了这个  


    "worker": {
        "im": 10,
        "sms": 20,
        "mail": 10
    },

对比上下文，我感觉这可能就是发微信，短信，邮件的worker数，不知道代码做没做判断，我直接把worker数改成了0，重启  

查看redis  

```
进到cli
redis-cli 

keys *

lrange /sms 0 0

 
```

看到有报警信息  跟0.1版本一模一样  我的脚本都不用修改，真是完美。

# 新系统测试

模拟报警发送消息。略

# 各机器agent更新

- ./falcon-agent 二进制文件更新 
- 配置文件更新（记得server地址更改）

写好 ansible role直接执行，
# nginx转发更改

此dashboard是使用的flask+gunicron 起的 我在内网nginx上做反向代理转发到这台机器8080 一直502，不得其解，因为着急验收，我直接在falcon_plus这台云主机上yum安装了一个nginx，然后内网nginx请求转发到这台机器80，就OK了。原因有时间再去看吧。。。或者说下次遇到falsk再看吧。。。


# 总结

&emsp;&emsp;遇到问题解决问题，不要思考的过于多导致自己迟迟不动手。

