---
title: rpm包制作
date: 2017-06-16 12:07:02
tags: rpm 
categories: 基础运维
---

---------
>今天突然收到老大发的这篇文章[分分钟拯救监控知识体系][1]，看到硬件监控的时候。突然想到我们是用的我们的硬件监控只是用了Dell的megacli工具监控了raid和磁盘的状态，连CPU温度主板温度这些基本指标好像都没有。因为我们用的open-falcon于是google下看到这篇[open-falconHWcheck][2] 


  [1]: http://mp.weixin.qq.com/s/TnhE_4afl0valv41V5ZFDA
  [2]: https://github.com/51web/hwcheck
  
  [toc]
  
  ----------------
  
## 制作过程
```
 git clone https://github.com/51web/hwcheck hwcheck-0.2
tar czf hwcheck-0.2.tar.gz hwcheck-0.2
rpmbuild -tb hwcheck-0.2.tar.gz
  ```
  具体可以使用 rpm --help 查看帮助

## rpm包相关操作
  
### RPM包安装：
```
rpm -ivh example.rpm
```
  
### 查看已经安装的RPM包
```
rpm -qa 
rpm -qa | grep tomcat4 查看 tomcat4 是否被安装；
```

### 验证RPM包
> 这个可用作系统有问题的时候或者是有人恶意更改系统文件的时候

```
[root@md5 lib]# rpm -Vf /etc/*
S.5....T.  c /etc/sysctl.conf
.......T.  c /etc/bashrc
S.5....T.  c /etc/hosts.allow
S.5....T.  c /etc/hosts.deny
```
解释下上边的命令  
其中，**S** 表示文件大小修改过，**T** 表示文件日期修改过。具体看man page
