---
title: shell脚本检测硬盘
date: 2017-06-25 09:53:32
tags: shell, 脚本
categories: shell
---

> 这个脚本是依赖dell提供的megacli工具写的，使用了一段时间，但是之后就使用了falcon的硬件监控工具，包括主板，风扇，硬盘，raid卡等比这个方便。

```
#!/bin/sh
allhosts="host01 "

log_dir=/home/chunyu_sys/disklog/
log_name=_raid_disk_monitor
logtime=$(date +%Y%m%d --date='1 days ago')
fix=.log

for i in $allhosts;do

host=`ssh $i "hostname"`
echo  "Checking RAID status on $host" >> $log_dir$logtime$log_name$fix
echo -e "\033[31m $host \033[0m"
echo "$host" >>$log_dir$logtime$log_name$fix
RAID_Contrller=`ssh $i 'megacli -AdpAllInfo -aALL |grep "Product Name" | cut -d: -f2'`
echo "Controller : $RAID_Contrller" >> $log_dir$logtime$log_name$fix

Online_disk_num=`ssh $i 'megacli  -PDList -aALL | grep Online | wc -l'`

echo "Totall number of Physical disks online : $Online_disk_num" >> $log_dir$logtime$log_name$fix
Degrade_disk=`ssh $i 'megacli -AdpAllInfo -a0 |grep "Degrade"'`
echo  "$Degrade_disk"  >>$log_dir$logtime$log_name$fix
Failed_disk=`ssh $i 'megacli -AdpAllInfo -a0 |grep "Failed Disks"'`
echo "$Failed_disk " >>$log_dir$logtime$log_name$fix

#Failed_disk_num=`ssh $i 'echo $Failed_disk |cut -d " " -f4'`
done
```
