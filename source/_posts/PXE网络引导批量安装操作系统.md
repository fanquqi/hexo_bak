---
title: PXE网络引导批量安装操作系统
date: 2016-08-03 15:49:00
tags: PXE
categories: 基础运维
---

## 前言：
       一般系统安装方式基本上都是做一个安装盘，然后每个机器都是插上U盘花上20分钟左右时间操作，包括选择引导方式，语言，键盘布局，设定初始root密码等一系列操作。较为繁琐，但是辛苦不是最大的问题，效率才是。
       这次我们机房迁移一共有三十台机器，每台不算初始化时间，只是安装系统就得很长时间，所以我们选择用PXE网络引导的方式，告别U盘。

## PXE技术分析：
### 工作过程
![](http://or2jd66dq.bkt.clouddn.com/pxe%E5%BC%95%E5%AF%BC%E6%B5%81%E7%A8%8B.gif)
其实就是DHCP+TFTP+FTP三个服务配合操作（FTP也可以用HTTP服务代替 因为我们本次操作的机器HTTP端口被占用所以直接选择FTP）



## 原理：
&emsp;&emsp;客户端PXE网卡启动
从DHCP服务器获得IP
从TFTP服务器上下载pxelinux.0、default
根据配置文件default指定的vmlinuz、initrd.img启动系统内核,并下载指定的ks.cfg文件
跟据ks.cfg去(HTTP/FTP/NFS)服务器下载RPM包并安装系统
完成安装

## 操作步骤：

DHCP服务器安装配置 
首先 确认DHCP服务没有安装，

```
rpm -qa | grep dhcp
```
    
没有安装过就直接yum安装（也要注意同一内网之中有没有别的DHCP服务在运行，避免冲突）
    
```
yum install dhcp*
```
    
然后配置一下 
    
```
vim /etc/dhcp/dhcpd.conf
```

<img src="http://or2jd66dq.bkt.clouddn.com/dhcp_conf.png" width="500" height="400" alt="dhcp_conf"/>

配置完直接重启 dhcp服务。

    service dhcp restart
但是我们会看到
![servie dhcpd start](http://or2jd66dq.bkt.clouddn.com/dhcp_start.png)

这个时候直接运行
    /usr/sbin/dhcpd 
就OK了(ps -ef 确认下进程是否在)
![dhcpd状态](http://or2jd66dq.bkt.clouddn.com/ps_dhcp.png)

## TFTP服务安装
首先确定机器有没有安装TFTP服务
    rpm -qa | grep tftp
如果没有直接yum安装
    yum install tftp*
然后稍微修改些配置
    disable 由yes改为no
    server_args = 加上-u nobody (用户可以是所有人)

/usr/lib/tftpboot 即为文件目录

![tftpd.conf](http://or2jd66dq.bkt.clouddn.com/modify_tftp.png)

服务启动
    systemctl start tftp


## FTP服务安装
确定ftp服务之前没有安装 

    yum -y install vsftpd 

修改一些配置   

    vim /etc/vsftpd/vsftpd.conf   

更改
    anonymous_enable=YES

这里需要保证anonymous_enable=YES （匿名用户登录开启）
然后测试ftp服务正常与否    可以找一台内网机器或者本机实验
![](http://or2jd66dq.bkt.clouddn.com/test_ftp.png)
如果能用anonymous 免密码登录到ftp 证明服务正常

## 文件拷贝         
&emsp;&emsp;现在基本服务已经搭建完成了  我们需要准备一下镜像文件等挂载到TFTP 和 FTP服务器上供网络引导的机器读取下载。因为我们上边dhcp.conf中已经写到了 pxe client 要向 TFTP服务器请求的filename pxelinux.0 
所以我们需要把这个文件拷贝到TFTP文件目录下。
PXE启动映像文件由syslinux提供，我们只要安装syslinux，就会生成一个pxelinux.0文件，

1. 只需要将 pxelinux.0 这个文件复制到TFTP根目录即可。

        yum install -y syslinux
        cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/

2. 把我们之前下载好的镜像ISO文件挂载一下，目标路径为ftp服务器文件目录位置

        mount -o loop /soft/CentOS-7-x86_64-Everything-1511.iso /var/ftp/pub/

3. 复制iso 镜像中的/image/pxeboot/initrd.img 和vmlinux 至/var/lib/tftpboot/ 文件夹中

        cp /var/ftp/pub/image/pxeboot/initrd.img /var/lib/tftpboot/
        cp /var/ftp/pub/image/pxeboot/vmlinux /var/lib/tftpboot/

4. 复制iso 镜像中的/isolinux/*.msg 至/var/lib/tftpboot/ 文件夹中
    
        cp /var/ftp/pub/isolinux/*.msg /var/lib/tftpboot/

5. 在/var/lib/tftpboot/ 中新建一个pxelinux.cfg目录
        
        mkdir /var/lib/tftpboot/pxelinux.cfg

6. 将iso 镜像中的/isolinux 目录中的isolinux.cfg复制到pxelinux.cfg目录中，同时更改文件名称为default
        
        cp /var/ftp/pub/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default

7. 修改default文件
        
        vim /var/lib/tftpboot/pxelinux.cfg/default

8. 服务启动
        
        systemctl start vsftpd


![kickstart_modify](http://or2jd66dq.bkt.clouddn.com/kiclstart_modify.png)
(如果有kickstart脚本也要在这里说明路径 例如: ks=ftp://10.215.33.12/pub/ks.cfg)

至此文件拷贝完成（别忘了重启三个服务DHCP，TFTP，FTP）
网络引导已经配置OK 重启客户端服务器的时候只需要进到boot manager中选择pxe网络引导即可。 

kickstart脚本测试完成后会附上

