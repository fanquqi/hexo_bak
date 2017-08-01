---
title: lsof十几个实例
date: 2017-06-06 16:36:41
tags: lsof
categories: 运维工具
---
# lsof十几个示例

-------
>lsof的意思是’列出打开的文件’，用于找出哪些文件被哪些进程打开或是占用。我们都知道Linux/UNIX的理念就是一切皆文件(包括pipes管道、sockets、directories目录、devices设备等等)。使用lsof命令的原因之一就是，当一个磁盘不能被卸载时，借助lsof这个命令我们可以轻易的识别哪些文件正在被占用.

----------
[toc]

---------
## 1.通过lsof命令列出所有打开的文件

>在下面的例子中，它会以长列表的形式显示打开的文件，为了便于理解，它以Command、PID、USER、FD、TYPE分类
``` powershell
(July) [root@blog local]# lsof
COMMAND     PID   TID           USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd       1                 root  cwd       DIR              253,0      4096        128 /
systemd       1                 root  rtd       DIR              253,0      4096        128 /
systemd       1                 root  txt       REG              253,0   1478168     198856 /usr/lib/systemd/systemd
systemd       1                 root  mem       REG              253,0     20032   50421307 /usr/lib64/libuuid.so.1.3.0
systemd       1                 root  mem       REG              253,0    252704   50886702 /usr/lib64/libblkid.so.1.1.0
systemd       1                 root  mem       REG              253,0     90664   50421293 /usr/lib64/libz.so.1.2.7
systemd       1                 root  mem       REG              253,0    157424   50421256 /usr/lib64/liblzma.so.5.2.2
systemd       1                 root  mem       REG              253,0     19888   50421655 /usr/lib64/libattr.so.1.1.0
systemd       1                 root  mem       REG              253,0     19776   51932115 /usr/lib64/libdl-2.17.so
systemd       1                 root  mem       REG              253,0    398264   51548149 /usr/lib64/libpcre.so.1.2.0
systemd       1                 root  mem       REG              253,0   2118128   50333965 /usr/lib64/libc-2.17.so
```
若不指定条件默认将显示所有进程打开的所有文件,lsof输出各列信息的意义如下:
- COMMAND：进程的名称
- PID：进程标识符
- USER：进程所有者
- FD：文件描述符，应用程序通过文件描述符识别该文件。如cwd、txt等
    - cwd 表示应用程序的当前工作目录
    - RTD 根目录
    - txt txt类型文件是程序代码，应用程序二进制文件本身或共享库
    - MEM 内存映射文件
    - u 表示该文件被打开并处于读取/写入模式，而不是只读 ® 或只写 (w) 模式。
    - W 表示该应用程序具有对整个文件的写锁。该文件描述符用于确保每次只能打开一个应用程序实例。
    - R 读访问
    - 初始打开每个应用程序时，都具有三个文件描述符，从 0 到 2，分别表示标准输入、输出和错误流。所以大多数应用程序所打开的文件的FD都是从3开始。
- TYPE：文件类型，如DIR、REG等
    - DIR 目录
    - REG 基本文件
    - CHR 字符特殊文件
    - FIFO 先进先出
    - UNIX unix域套接字
- DEVICE：指定磁盘的名称
- SIZE：文件的大小
- NODE：索引节点（文件在磁盘上的标识）
- NAME：打开文件的确切名称

----------------
# 2.列出特定用户打开的文件

使用**-u**选项后接用户指定某个用户打开文件
``` powershell
# lsof -u apache
COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
httpd   6032 apache  cwd    DIR    8,3     4096       2 /
httpd   6032 apache  rtd    DIR    8,3     4096       2 /
httpd   6032 apache  txt    REG    8,3   354688 1605148 /usr/sbin/httpd
httpd   6032 apache  mem    REG    8,3    65928  654110 /lib64/libnss_files-2.12.so
```

---------
# 3.查找特定端口运行的进程
使用**-i**选项来查找正在运行特定端口的进程
``` powershell
# lsof -i TCP:53
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
named   16885 named   20u  IPv4  61664      0t0  TCP localhost:domain (LISTEN)
# lsof -i UDP:53
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
named   16885 named  512u  IPv4  61663      0t0  UDP localhost:domain
# lsof -i:53
named   16885 named   20u  IPv4  61664      0t0  TCP localhost:domain (LISTEN)
named   16885 named  512u  IPv4  61663      0t0  UDP localhost:domain
```

--------

# 4.列出ipv4 和ipv6的文件
``` powershell
 #lsof -i 4
COMMAND    PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd      1239  root    3u  IPv4  10081      0t0  TCP *:ssh (LISTEN)
# lsof -i 6
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd     1239   root    4u  IPv6  10083      0t0  TCP *:ssh (LISTEN)
```
-------
#5.列出TCP端口范围1-1024端口
``` powershell
# lsof -i TCP:1-1024
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd     1239   root    3u  IPv4  10081      0t0  TCP *:ssh (LISTEN)
sshd     1239   root    4u  IPv6  10083      0t0  TCP *:ssh (LISTEN)
httpd    2142   root    4u  IPv6  13337      0t0  TCP *:http (LISTEN)
```

-----
# 6.通过脱字符排除某个用户
``` powershell

# lsof -u^root
COMMAND     PID   USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
dbus-daem  1212   dbus  cwd    DIR                8,3     4096       2 /
dbus-daem  1212   dbus  rtd    DIR                8,3     4096       2 /
```
---------

# 7.查找特定文件用户和命令
``` powershell
# lsof -i -u apache 
COMMAND    PID   USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
httpd     6032 apache  txt    REG    8,3   354688 1605148 /usr/sbin/httpd
httpd     6032 apache  mem    REG    8,3     9488  271645 /usr/lib64/apr-util-1/apr_ldap-1.so
```
-------

# 8.查看所有的网络连接
``` powershell
# lsof -i
COMMAND    PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd      1239   root    3u  IPv4  10081      0t0  TCP *:ssh (LISTEN)
sshd      1239   root    4u  IPv6  10083      0t0  TCP *:ssh (LISTEN)
```
-------

# 9.采用pid搜索
``` powershell
# lsof -p 1
COMMAND PID USER   FD   TYPE             DEVICE SIZE/OFF   NODE NAME
init      1 root  cwd    DIR                8,3     4096      2 /
init      1 root  rtd    DIR                8,3     4096      2 /
init      1 root  txt    REG                8,3   150352 527181 /sbin/init

```
-------

# 10.杀死某个特定用户的所有活动
``` powershell
# kill -9 `lsof -t -u named`

```
-------

# 11.查看谁在使用文件系统,在卸载文件系统时
``` powershell
# lsof /mnt/
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
bash    16672 root  cwd    DIR   11,0     8192 1856 /mnt
lsof    17041 root  cwd    DIR   11,0     8192 1856 /mnt
lsof    17042 root  cwd    DIR   11,0     8192 1856 /mnt

```
--------

# 12.查看被删除的文件
``` powershell
# lsof | grep deleted --color
console-k  1291   root  txt       REG                8,3   155008    1577669 /usr/sbin/console-kit-daemon.#prelink#.bXthE2 (deleted)
tail      17553   root    3r      REG                8,3        6     523317 /tmp/test2 (deleted)

```

# 13.恢复误删除文件
>当Linux计算机受到入侵时，常见的情况是日志文件被删除，以掩盖攻击者的踪迹。管理错误也可能导致意外删除重要的文件，比如在清理旧日志时，意外地删除了数据库的活动事务日志。有时可以通过lsof来恢复这些文件。

>当进程打开了某个文件时，只要该进程保持打开该文件，即使将其删除，它依然存在于磁盘中。这意味着，进程并不知道文件已经被删除，它仍然可以向打开该文件时提供给它的文件描述符进行读取和写入。除了该进程之外，这个文件是不可见的，因为已经删除了其相应的目录索引节点。
>在/proc 目录下，其中包含了反映内核和进程树的各种文件。/proc目录挂载的是在内存中所映射的一块区域，所以这些文件和目录并不存在于磁盘中，因此当我们对这些文件进行读取和写入时，实际上是在从内存中获取相关信息。大多数与 lsof 相关的信息都存储于以进程的 PID 命名的目录中，即/proc/1234 中包含的是PID为1234 的进程的信息。

当系统中的某个文件被意外地删除了，只要这个时候系统中还有进程正在访问该文件，那么我们就可以通过lsof从/proc目录下恢复该文件的内容。 假如由于误操作将/var/log/messages文件删除掉了，那么这时要将/var/log/messages文件恢复的方法如下：

- 首先使用lsof来查看当前是否有进程打开/var/logmessages文件，如下:
``` powershell
# lsof |grep /var/log/messages
syslogd   1283      root    2w      REG        3,3  5381017    1773647 /var/log/messages (deleted)
```
- 从上面的信息可以看到 PID 1283（syslogd）打开文件的文件描述符为 2。同时还可以看到/var/log/messages已经标记被删除了。因此我们可以在 /proc/1283/fd/2 （fd下的每个以数字命名的文件表示进程对应的文件描述符）中查看相应的信息，如下：
``` powershell
# head -n 10 /proc/1283/fd/2
Aug  4 13:50:15 holmes86 syslogd 1.4.1: restart.
Aug  4 13:50:15 holmes86 kernel: klogd 1.4.1, log source = /proc/kmsg started.
Aug  4 13:50:15 holmes86 kernel: Linux version 2.6.22.1-8 (root@everestbuilder.linux-ren.org) (gcc version 4.2.0) #1 SMP Wed Jul 18 11:18:32 EDT 2007
Aug  4 13:50:15 holmes86 kernel: BIOS-provided physical RAM map:
Aug  4 13:50:15 holmes86 kernel:  BIOS-e820: 0000000000000000 - 000000000009f000 (usable)
Aug  4 13:50:15 holmes86 kernel:  BIOS-e820: 000000000009f000 - 00000000000a0000 (reserved)
Aug  4 13:50:15 holmes86 kernel:  BIOS-e820: 0000000000100000 - 000000001f7d3800 (usable)
Aug  4 13:50:15 holmes86 kernel:  BIOS-e820: 000000001f7d3800 - 0000000020000000 (reserved)
Aug  4 13:50:15 holmes86 kernel:  BIOS-e820: 00000000e0000000 - 00000000f0007000 (reserved)
Aug  4 13:50:15 holmes86 kernel:  BIOS-e820: 00000000f0008000 - 00000000f000c000 (reserved)
```

- 从上面的信息可以看出，查看 /proc/1283/fd/2 就可以得到所要恢复的数据。如果可以通过文件描述符查看相应的数据，那么就可以使用 I/O 重定向将其复制到文件中，如:
``` powershell
cat /proc/1283/fd/2 > /var/log/messages

```
