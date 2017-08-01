---
title: 记录主机history
date: 2017-07-12 14:10:20
tags: bash
---
> 起因，有个哥们儿要离职，直接上线上把他机器训练的东西拷贝到电脑本地，我们用的vpn，有些服务对此有点依赖，不详细说，总之影响到了一丢丢线上的情况，所以CTO很不高兴，机器历史记录也没有，啥都没有，多亏他承认了。但是我这个运维还是多少显得有点尴尬。

下面说一些改进措施
- 不准任何人直接登录线上机器，必须通过跳板机（上传现在文件也必须通过跳板机加以控制）
- openvpn与线上带宽解耦
- openvpn限速，线上加iptables阻止openvpn的地址，只接受跳板机(加一条运维通道，永远有B方案)
- 历史记录需要详细记录

前三条很好就解决了。
下面内容详细记录下第四条的实现方法。

bash是多数Linux发行版默认的shell，虽然不及zsh好用，但比其它的shell好太多。
我们的生产服务器很多，没有用跳板机，又是多人共用root用户，为了审计用户操作，需要记录执行命令的用户、时间和ip等信息。本文之所以要优化，主要是因为bash默认配置存在以下几点不足：

1. 历史记录保存数目有限，默认1000条

2. 记录不详细，不记录命令执行时间/执行用户名/用户ip等

3. 历史记录会丢失，主要有两种情况：
    1. bash异常退出 
    2. 同一用户多处登录或开了多个会话，只会记录最后退出的会话历史

所以我决心自己记录下`bash_history` 并把它写到ES之中[传送门](https://fanquqi.github.io/2017/06/12/file-beat%E6%8E%A5%E5%85%A5ELK/),这样有人做了什么非法的事情，即使他清空了历史记录我的ES中也能存着他的罪证，除非他每条history都秒删，在速度上超过filebeat的读取的速度，事实证明不怎么可能。

## 常规rsyslog实现
> [参考链接](http://www.361way.com/history-log-audit/4147.html)
### 配置全局bash历史记录格式
在/etc/bashrc中写入
```
export PROMPT_COMMAND='RETRN_VAL=$?;logger -p local6.debug "$(who am i) [$$]: $(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//" ) [$RETRN_VAL]"'
```


### 配置rsyslog
新增文件/etc/rsyslog.d/bash.conf,内容
```
local6.*    /var/log/bash_history.log
```

### 重启rsyslogd
```
systemctl restart rsyslogd
```

### ansible 脚本
```
---

#- name: add scripts to bashrc
#  lineinfile:
#    dest=/etc/bashrc
#    line={{item}}
#  with_items: '{{bashrc_line}}'
#  register: profile
- name: copy file to /etc
  copy: src=bash_log.conf dest=/var/tmp

- name: echo to /etc/bashrc
  shell: cat /var/tmp/bash_log.conf >> /etc/bashrc
  register: bashrc

- name: source file
  shell: source /etc/bashrc
  when: bashrc.changed


- name: copy bash.conf to /etc/rsyslog.d
  copy: src=bash.conf dest=/etc/rsyslog.d
  register: rsyslog_conf

- name: restart rsyslog
  service: name=rsyslog.service state=restarted
  when: rsyslog_conf.changed
```
刷到每个机器上就好了   注意最好配置下logrotate每天切割一下

这样就可以在kibana中看到每个登录人员的操作情况了。

## 查看用户痕迹过程展示

上面中控机上是每个人对应一个自己名字拼音的用户，使用此用户跳到线上机器，但是测试环境是直接本地可以连，测试被人搞坏了进度delay怎么办？也需要查证，可以使用以下方法
![](http://or2jd66dq.bkt.clouddn.com/bash_history_modify.png)
