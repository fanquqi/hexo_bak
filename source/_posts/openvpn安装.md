---
title: openvpn安装
date: 2017-06-25 09:58:40
tags: openvpn
categories: 基础运维
---

> openvpn不多作介绍，直接上部署过程

# 服务器环境
* 机器名：host01
* 操作系统：CentOS Linux release 7.0.1406 (Core)
* 内网IP：10.******
* 外网IP：12.******
* 安装方式：yum
* openvpn版本：OpenVPN 2.3.12 x86_64-redhat-linux-gnu

# 安装步骤

## 安装前操作
 
`关闭selinux 配置防火墙`

```    
 setenforce 0
 sed -i '/^SELINUX=/c\SELINUX=disabled' /etc/selinux/config
 iptables -I INPUT -p udp --dport 1194 -m comment --comment "openvpn" -j ACCEPT
 iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j MASQUERADE 
 service iptables save
                       
```

`开启路由转发功能`
```
 sed -i '/net.ipv4.ip_forward/s/0/1/' /etc/sysctl.conf
 echo "1">/proc/sys/net/ipv4/ip_forward
 sysctl -p
```
`安装openssl，lzo`（用于压缩通讯数据，加快传输速度）
``` 
yum install openssl openssl-delvel
yum install lzo
```

## 安装步骤

`安装配置openvpn和easy-rsa`

```
yum install openvpn easy-rsa
```

`修改vars文件`

```
cat /usr/share/easy-rsa/2.0/vars | grep -Ev "^$|#"
export EASY_RSA="`pwd`"
export OPENSSL="openssl"
export PKCS11TOOL="pkcs11-tool"
export GREP="grep"
export KEY_CONFIG=`$EASY_RSA/whichopensslcnf $EASY_RSA`
export KEY_DIR="$EASY_RSA/keys"
echo NOTE: If you run ./clean-all, I will be doing a rm -rf on $KEY_DIR
export PKCS11_MODULE_PATH="dummy"
export PKCS11_PIN="dummy"
export KEY_SIZE=2048
export CA_EXPIRE=3650
export KEY_EXPIRE=3650
export KEY_COUNTRY="CN"
export KEY_PROVINCE="CA"
export KEY_CITY="Bei Jing"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"
export KEY_NAME="EasyRSA"       

**copy easy_rsa目录**

cp -r /usr/share/easy-rsa/2.0/* /etc/openvpn/
```


初始化环境变量
```
source vars
```

清除keys目录下所有与证书相关的文件

```
./clean-all
```

生成根证书ca.crt 根秘钥ca.key（一路回车）

```
./build-ca
```

为服务端生成证书秘钥（一路回车）

```
./build-key-server server
```

创建迪菲·赫尔曼密钥，会生成dh2048.pem文件（过程比较慢）

```
./build-dh
```

生成ta.key（防DOS攻击，等）

```
openvpn --genkey --secret keys/ta.key
```

创建服务端配置文件

```
在openvpn目录下创建一个keys目录
mkdir /etc/openvpn/keys
```

复制一份刚创建好的证书秘钥到新创建的keys

```
cp /usr/share/easy-rsa/2.0/keys/{ca.crt,server.{crt,key},dh2048.pem,ta.key} /etc/openvpn/keys/
```

复制一份配置文件模板到/etc/openvpn/

```
cp /usr/share/doc/openvpn-2.3.12/sample/sample-config-files/server.conf /etc/openvpn/
```

修改一下配置文件(这里使用的udp,会比tcp更快一下)
```
[root@vpn ~]# cat /etc/openvpn/server.conf | grep -Ev "^#|;|^$"
port 1194
proto udp
dev tun
ca keys/ca.crt
cert keys/server.crt
key keys/server.key  # This file should be kept secret
dh keys/dh2048.pem
server 10.8.3.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.0.0.0 255.0.0.0"
keepalive 10 120
tls-auth keys/ta.key 0 # This file is secret
comp-lzo
persist-key
persist-tun
status openvpn-status.log
log         openvpn.log
verb 5
```

## openvpn启动 
systemctl -f enable openvpn@server.service
systemctl start openvpn@server.service

## 添加|删除用户
**添加用户脚本**(可以使用它自动添加用户)
```
#!/bin/bash
vpnServer1=10.9.104.39
#vpnServer2=10.12.1.28
# a. 在vpnserver01中创建新vpn用户
if [ -z $1 ]
then
  echo "Error:请在脚本后添加用户名作为参数，例如：'./01_addUser.sh zhangsan'"
else
  cp /etc/openvpn/keys/$1.crt ./ > /dev/null 2>&1
  if [ -f $1.crt ]
  then
    echo "$1 用户已存在，请检查！"
    rm -f *.crt
  else

    cd /etc/openvpn/ && pwd && source /etc/openvpn/vars && ./build-key --batch $1
#    cd /etc/openvpn/ && pwd && source /usr/share/easy-rsa/2.0/vars && ./build-key --batch $1
    # b. 新用户配置文件修改以及打包发送mail到用户
    cd -
    mkdir -p $1
cat << EOF >> ./$1/$1.ovpn
client
dev tun
proto udp
remote ****** 1194
remote-random
resolv-retry 10
nobind
persist-key
persist-tun
ca ca.crt
cert $1.crt
key $1.key
remote-cert-tls server
tls-auth ta.key 1
comp-lzo
verb 3
EOF

    cp /etc/openvpn/keys/{ca.crt,$1.{crt,key},ta.key} ./$1/

#    cp /etc/openvpn/keys/$1.{crt,key} /home/chunyu_sys/workspace/cy_ansible/roles/vpn_agent/files/
    cp README.txt ./$1/;cp openVPN-clinet-config-for-Mac.pdf ./$1/
    cp /etc/hosts ./$1/chunyu_hosts
    tar czf $1.tar.gz $1/
    rm -fr $1/
#    python ./send_mail.py $1@chunyu.me "[运维][vpn申请]openVPN configuration files" $1.tar.gz
#    mutt -s "openVPN configuration files" -a $1.tar.gz -- xiepengcheng@chunyu.me < $1.tar.gz
    #rm -f $1.tar.gz
    echo "用户 $1 创建完毕"
  fi
fi
```

**删除用户脚本**

```
#!/bin/bash
vpnServer1=host01

if [ -z $1 ]
then
  echo "Error:请在脚本后添加用户名作为参数，例如：'./02_delUser.sh zhangsan'"
else
  cp /etc/openvpn/keys/$1.crt ./
  if [ -f $1.crt ]
  then
  cd /etc/openvpn && source vars && ./revoke-full $1
  rm -rf /etc/openvpn/keys/$1.*
  echo "用户 $1 删除完毕"
  cd -
  rm -f $1.crt
  else
    echo "$1 此VPN用户不存在，请检查!"
  fi
fi
```


## 使用TC进行限速 
因为大家都不遵守规则，总是把vpn带宽占满，影响别的用户使用，所以必须加以限制

```
tc qdisc add dev tun0 root handle 1:0 htb default 10

tc class add dev tun0 parent 1:0 classid 1:1 htb rate 10Mbit burst 15k

tc class add dev tun0 parent 1:1 classid 1:10 htb rate 640kbit ceil 640kbit burst 15k

tc qdisc add dev tun0 parent 1:10 handle 10: sfq perturb 10

tc filter add dev tun0 protocol ip parent 1:0 prio 3 u32 match ip dst 10.0.0.6 flowid 1:10

```

上面规则可以控制10.0.0.6这个用户的下载带宽为:80KB/s,以此类推限制其它用户.如果会写shell,可以作一个程序,加到openvpn的拨入脚本中.



上传是在eth0(假如外网卡是eth0)上做.

总结：
如果iptables关了，openvpn服务起了，客户端还是连不上，telnet 1194端口也不通，记得把云主机外网防火墙改一下。
