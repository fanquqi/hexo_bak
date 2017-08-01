---
title: shell脚本自动安装zabbix_agent
date: 2017-06-25 09:43:10
tags: shell, 脚本
categories: shell
---
> 现在公司都用ansible替换zabbix了，之前写的安装zabbix_agent脚本，在这儿记录一下，以备不时之需。

`zabbix_agent_install.sh`
```
#!/usr/bin/bash
#allhosts="mysql1 mysql2 mysql3"
#可以执行远端命令
for host in $allhosts
do

echo $host
ssh root@$host "mkdir -p /usr/local/zabbix/etc/"
ssh root@$host "mkdir -p /root/soft/"
ssh root@$host "mkdir -p /var/log/zabbix/"

sed -i "s/unknown/$host/g" ./zabbix_agentd.conf

cat ./zabbix_agentd.conf


scp ./zabbix_agentd.conf root@$host:/usr/local/zabbix/etc/

sed -i "s/$host/unknown/g" ./zabbix_agentd.conf


scp ./zabbix-2.4.2.tar.gz root@$host:/root/soft/
scp ./install_zabbix_agent_impl.sh root@$host:/root/soft/
ssh root@$host "bash /root/soft/install_zabbix_agent_impl.sh"


done
```

`zabbix_agent_install_impl.sh`
```
#!/usr/bin/bash

cd /root/soft
tar zxvf ./zabbix-2.4.2.tar.gz
cd /root/soft/zabbix-2.4.2
./configure --prefix=/usr/local/zabbix --enable-agent
make;make install
groupadd zabbix
useradd -g zabbix -s /sbin/nologin zabbix
chown zabbix:zabbix /var/log/zabbix
chown zabbix:zabbix /var/log/zabbix
/usr/local/zabbix/sbin/zabbix_agentd -c /usr/local/zabbix/etc/zabbix_agentd.conf
echo "/usr/local/zabbix/sbin/zabbix_agentd -c /usr/local/zabbix/etc/zabbix_agentd.conf" >>/etc/rc.local
```

`zabbix_agent.conf`

```
PidFile=/var/log/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
#server 为zabbix_server地址
Server=192.168.0.1
ListenPort=10050
ServerActive=192.168.0.1
Hostname=unknown
#HostnameItem=system.hostname
RefreshActiveChecks=60
BufferSize=1024
UnsafeUserParameters=1
Include=/usr/local/zabbix/etc/zabbix_agentd.conf.d/*.conf
Timeout=20
AllowRoot=1
```

记得把安装包也放在脚本目录下
