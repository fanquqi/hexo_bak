---
title: 通过ucloud_API查看资源价格
date: 2017-07-11 17:34:50
tags: python 
categories: python
---

> 我们公司有些服务使用ucloud主机，付费等问题都是我负责，前一段时间跟他们的架构沟通了一下，发现他们的API还是很方便的，尤其他们提供弹性IP可以自由伸缩，我们完全可以写个脚本统计带宽，实时调整。如果在云上跑docker，完全可以直接通过流量挥着访问量实时业务扩容。运维真的做到自动化。

    ucloud提供了官方的`apk`[链接](https://github.com/ucloud-web/python-sdk-v2.git)
有的同学直接把它写成了python命令 [链接](https://pypi.python.org/pypi/ucli/)

本来只是想做一个统计价格的工具，写着写着，就像反正收集实例信息还不如都挨个统计一下展示出来，
因为：

1. ucloud信息展示的并不清晰  
2. 如果看到价格有异常肯定第一时间想知道到底哪个贵些，贵在哪里，更方便直观一些

我在下边主要展示一下自己手写的几个脚本通过API获取实例信息，再通过实例信息获取实例价格，最后统一发送到influxDB中去到grafana上呈现。定时每天跑一次。代码水平有限，大家请不要嘲笑。


先来看下他的apk和配置文件
他的public_key 和 private_key 涉及到自己独特的加秘方式。可以去ucloud官网看看 


## `apk.py`

```
# -*- coding: utf-8 -*-
import hashlib, json, httplib
import urlparse
import urllib
import sys
from config import *


class UCLOUDException(Exception):
    def __str__(self):
        return "Error"


def _verfy_ac(private_key, params):
    items = params.items()
    items.sort()

    params_data = ""
    for key, value in items:
        params_data = params_data + str(key) + str(value)

    params_data = params_data+private_key

    '''use sha1 to encode keys'''
    hash_new = hashlib.sha1()
    hash_new.update(params_data)
    hash_value = hash_new.hexdigest()
    return hash_value


class UConnection(object):
    def __init__(self, base_url):
        self.base_url = base_url
        o = urlparse.urlsplit(base_url)
        if o.scheme == 'https':
            self.conn = httplib.HTTPSConnection(o.netloc)
        else:
            self.conn = httplib.HTTPConnection(o.netloc)

    def __del__(self):
        self.conn.close()

    def get(self, resouse, params):
        resouse += "?" + urllib.urlencode(params)
        print("%s%s" % (self.base_url, resouse))
        self.conn.request("GET", resouse)
        response = json.loads(self.conn.getresponse().read())
        return response


class UcloudApiClient(object):
    # 添加 设置 数据中心和  zone 参数
    def __init__(self, base_url, public_key, private_key):
        self.g_params = {}
        self.g_params['PublicKey'] = public_key
        self.private_key = private_keyurl
        self.conn = UConnection(base_url)

    def get(self, uri, params):
        # print params
        _params = dict(self.g_params, **params)

        if project_id :
            _params["ProjectId"] = project_id

        _params["Signature"] = _verfy_ac(self.private_key, _params)
        return self.conn.get(uri, _params)

```

## `conf.py`
```
#-*- encoding: utf-8 -*-
#配置公私钥"""
public_key  = "at************************"
private_key = "e7***************"
#project_id = "******" # 项目ID 请在Dashbord 上获取

base_url    = "https://api.ucloud.cn"

```


## collect_uhost_price.py
> 通过查看机器示例，按付费方式查看机器价格。
具体查看[ucloud uhost API](https://docs.ucloud.cn/api/uhost-api/index)
```
#!/usr/bin/python
#-*- coding:utf-8 -*-

# author:fanquanqing
# collect ucloud uhost price
from sdk import UcloudApiClient
from config import *
from collections import Iterable
import sys
import json


#host_list = []

# 收集uhost信息
def collect_uhost_info(Regions_list,ProjectId):
    '''
    包含 hostname, ip, region, cpu, mem, diskspace, chargetype, count, ImageId, osname.
    '''
    host_info_list = []
    for Region in Regions_list:
        ApiClient = UcloudApiClient(base_url, public_key, private_key)
        Parameters={
        "Action":"DescribeUHostInstance",
        "Region":Region,
        "ProjectId":ProjectId,
        "Limit":"1000"
        }
        response = ApiClient.get("/", Parameters);
        length = response['TotalCount']
        for i in range(length):
            host_info = {}
            #ImageId = response["UHostSet"][i]["BasicImageId"].encode("utf-8")
            ChargeType = response["UHostSet"][i]["ChargeType"].encode("utf-8")
            #host_info["Action"] = "GetUHostInstancePrice"
            host_info["Region"] = Region
            host_info["ImageId"] = "uimage-kg0w4u"
            host_info["Hostname"] = response["UHostSet"][i]["Name"]
            host_info["CPU"] = response["UHostSet"][i]["CPU"]
            host_info["Memory"] = response["UHostSet"][i]["Memory"]

            if len(response["UHostSet"][i]["DiskSet"]) > 1:
                host_info["DiskSpace"] = response["UHostSet"][i]["DiskSet"][1]["Size"]
            else:
                host_info["DiskSpace"]=0
            host_info["Count"] = 1
            host_info["ChargeType"] = ChargeType
            host_info["OsName"] = response["UHostSet"][i]["OsName"].split()[0]
            host_info["loaclIP"] = response["UHostSet"][i]["IPSet"][0]["IP"]
            if len(response["UHostSet"][i]["IPSet"])>1:
                host_info["EIP"]=response["UHostSet"][i]["IPSet"][1]["IP"]
            else:
                host_info["EIP"]="none"

            host_info_list.append(host_info)
    #print host_info_list
    return host_info_list


#通过机器配置得到某一台机器价格
def get_uhost_price(host_instance_info):
    ApiClient = UcloudApiClient(base_url, public_key, private_key)
    Parameters= host_instance_info
    response = ApiClient.get("/", Parameters );
#    print json.dumps(response, sort_keys=True, indent=4, separators=(',', ': '))
    price = float(response["PriceSet"][0].values()[0])
    return price

#通过机器配置信息得到所有机器价格
def get_all_uhost_price(host_info_list):

    for host_info in host_info_list:
        host_params={}
        host_params["Action"]="GetUHostInstancePrice"
        host_params["ImageId"]=host_info["ImageId"]
        host_params["CPU"]=host_info["CPU"]
        host_params["Memory"]=host_info["Memory"]
        host_params["Count"]=host_info["Count"]
        host_params["DiskSpace"]=host_info["DiskSpace"]
        host_params["Region"]="cn-bj2"
        host_params["ChargeType"]=host_info["ChargeType"]
        if host_params["ChargeType"]=="Year":
            price = get_uhost_price(host_params)
        elif host_params["ChargeType"]=="Month":
            price = get_uhost_price(host_params)*12
        else:
            price=0
            print "有临时机器请排查。"
        host_info["Price"]=price

    return host_info_list

def collect_uhost_price(setting_info):

    host_info_total = []
    for Projectname in setting_info:
        for ProjectId in setting_info[Projectname]:
            Regions_list = setting_info[Projectname][ProjectId]
            host_info_list = collect_uhost_info(Regions_list,ProjectId)
            project_host_list = get_all_uhost_price(host_info_list)
            host_info_total.extend(project_host_list)
    return host_info_total



if __name__ == '__main__':
    host_info_list = collect_uhost_info(['cn-bj2','hk'],"org-oddm1w")
    host_price_list = get_all_uhost_price(host_info_list)
    #print host_price_list
```

## collect_eip_price.py

> 收集弹性IP信息
```
#!/usr/bin/env python
# -*- coding:utf-8 -*-
# author:fanquanqing
# collect ucloud eip price
from sdk import UcloudApiClient
from config import *
import sys
import json

# 得到所有的EIP实例信息
def get_eip_instance(Regions_list,ProjectId):
    eip_all = []
    for Region in Regions_list:
        ApiClient = UcloudApiClient(base_url, public_key, private_key)
        Parameters={"Action":"DescribeEIP", "Region":Region, "ProjectId":ProjectId}
        response = ApiClient.get("/", Parameters );
        for eip in response['EIPSet']:
            eip_instance = {}
            # 地域
            eip_instance['Region']=Region
            # IP
            eip_instance['IP']=eip['EIPAddr'][0]['IP']

            # 运营商线路
            eip_instance['OperatorName']=eip['EIPAddr'][0]['OperatorName'].encode('utf-8')
            # 带宽
            eip_instance['Bandwidth']=eip['Bandwidth']
            # 付费周期
            eip_instance['ChargeType']=eip['ChargeType'].encode('utf-8')
            # 付费方式(是否绑定共享带宽)
            eip_instance['PayMode']=eip['PayMode'].encode('utf-8')
            eip_all.append(eip_instance)

    return eip_all
    #print json.dumps(response, sort_keys=True, indent=4, separators=(',', ': '))

# 查看单个EIP实例价格
def get_eip_price(eip):
    ApiClient = UcloudApiClient(base_url, public_key, private_key)
    Parameters=eip
    response = ApiClient.get("/", Parameters );
    #print response
    price = response['PriceSet'][0]['Price']
    return price


# 获取所有EIP价格
def get_all_eip_price(eip_all):

    for eip_info in eip_all:
        eip_params={}
        eip_params['Action']="GetEIPPrice"
        eip_params['Region']=eip_info['Region']
        eip_params['OperatorName']=eip_info['OperatorName']
        eip_params['Bandwidth']=eip_info['Bandwidth']
        eip_params['ChargeType']=eip_info['ChargeType']
        eip_params['PayMode']=eip_info['PayMode']
        if eip_params['ChargeType']=="Year":
            price = get_eip_price(eip_params)
        elif eip_params['ChargeType']=="Month":
            price = get_eip_price(eip_params)*12
        else:
            price=0
            print "有临时EIP请排查"
        eip_info["Price"]=price
    return eip_all


def collect_eip_price(setting_info):
    eip_info_total = []
    for Projectname in setting_info:
        for ProjectId in setting_info[Projectname]:
            Regions_list = setting_info[Projectname][ProjectId]
            eip_all=get_eip_instance(Regions_list,ProjectId)
            price_all=get_all_eip_price(eip_all)
            eip_info_total.extend(price_all)
    #print eip_info_total
    return eip_info_total

if __name__ == '__main__':
    collect_eip_price({"chunyu":{"org-oddm1w":['cn-bj2','hk']}, "uhs":{"org-shbbct":["cn-bj2"]}})



```

## collect_udisk_price.py

> 收集云硬盘信息，这个只是取到实例自己手动算的价格
```
#!/usr/bin/env python
# -*- coding:utf-8 -*-

# author: fanquanqing
# 收集ucloud云硬盘信息 及价格

from sdk import UcloudApiClient
from config import *
import sys
import json

# 获取udisk 信息
def get_udisk_info(Regions_list,ProjectId):
    udisk_list = []
    for Region in Regions_list:
        ApiClient = UcloudApiClient(base_url, public_key, private_key)
        Parameters={"Action":"DescribeUDisk", "Region":Region, "ProjectId":ProjectId}
        response = ApiClient.get("/", Parameters );
        for udisk in response['DataSet']:
            udisk_dic = {}
            udisk_dic['Region'] = Region
            #udisk_dic['Action'] = "DescribeUDiskPrice"
            udisk_dic['Size'] = udisk['Size']
            udisk_dic['ChargeType'] = udisk['ChargeType'].encode('utf-8')
            udisk_dic['UHostName'] = udisk['UHostName']
            udisk_dic['Quantity'] = 1
            udisk_dic['Zone'] = "cn-bj2-02"
            udisk_list.append(udisk_dic)
    return udisk_list

# 通过udisk信息获取价格
def get_udisk_price(udisk_list):
    for udisk in udisk_list:
        ApiClient = UcloudApiClient(base_url, public_key, private_key)
        udisk_params={}
        udisk_params['Action']="DescribeUDiskPrice"
        udisk_params['Region']=udisk['Region']
        udisk_params['Size']=udisk['Size']
        udisk_params['ChargeType']=udisk['ChargeType']
        udisk_params['Quantity']=udisk['Quantity']
        udisk_params['Zone']=udisk['Zone']
        Parameters = udisk_params
        response = ApiClient.get("/", Parameters );
        if udisk_params['ChargeType']=="Year":
            price = response['DataSet'][0]['Price']/100
        elif udisk_params['ChargeType']=="Month":
            price = response['DataSet'][0]['Price']/10
        else:
            print "有付费方式异常的云硬盘，请排查。"
        udisk["Price"]=price
    return udisk_list

# 通过付费周期获取udisk价格
def collect_udisk_price(setting_info):
    udisk_price_info = []
    for Projectname in setting_info:
        for ProjectId in setting_info[Projectname]:
            Regions_list = setting_info[Projectname][ProjectId]
            udisk_list = get_udisk_info(Regions_list,ProjectId)
            udisk_price_info.extend(get_udisk_price(udisk_list))
    return udisk_price_info



if __name__ == '__main__':
    collect_udisk_price({"chunyu":{"org-oddm1w":['cn-bj2','hk']}, "uhs":{"org-shbbct":["cn-bj2"]}})

```

## collect_sharebandwidth_price.py
> 收集共享带宽信息,因为没有计算价格的API这个价格也是手动算的。

```
#!/usr/bin/python
# -*- coding: utf-8 -*-
#author fanquanqing
#收集共享带宽信息获取每年消费情况
from sdk import UcloudApiClient
from config import *
import sys
import json

def get_bandwidth_info(Regions_list,ProjectId):
    bandwidth = []
    for Region in Regions_list:

        ApiClient = UcloudApiClient(base_url, public_key, private_key)
        Parameters={"Action":"DescribeShareBandwidth","Region":Region,"ProjectId":ProjectId}
        response = ApiClient.get("/", Parameters );
        #print response
        for bw in response['DataSet']:
            bw_instance = {}
            bw_instance['ShareBandwidth'] =  bw['ShareBandwidth']
            bw_instance['ChargeType'] = bw['ChargeType']
            bw_instance['Name']=bw['Name']
            bandwidth.append(bw_instance)
    return bandwidth
#   print json.dumps(response, sort_keys=True, indent=4, separators=(',', ': '))

def get_price(bandwidth):
    '''
    价格计算说明:ucloud没有提供API查询共享带宽价格，所有的共享带宽价格都是90/M/月 月付*12,年付*10
    '''
    price_list = []
    for bw in bandwidth:
        if bw['ChargeType']=="Month":
            price = bw['ShareBandwidth']*90*12
            bw["Price"]=price
        else:
            price = bw['ShareBandwidrh']
            bw["Price"]=price
    return bandwidth

def collect_sharebandwidth_price(setting_info):
    bw_price_info = []
    for Projectname in setting_info:
        for ProjectId in setting_info[Projectname]:
            Regions_list = setting_info[Projectname][ProjectId]
            bw_info = get_bandwidth_info(Regions_list,ProjectId)
            bw_price_info.extend(get_price(bw_info))
    #print bw_price_info
    return bw_price_info


if __name__ == '__main__':
    collect_sharebandwidth_price({"chunyu":{"org-oddm1w":['cn-bj2','hk']}, "uhs":{"org-shbbct":["cn-bj2"]}})

```

## get_all_price.py

> 给各实例价格求和，发送到influxdb。可以每天跑一下cron更新下内容。**里面发送的influxDB是事先封装好的包直接导入的，并不是官方包**
```
#!/usr/bin/env python
# -*- coding:utf-8 -*-

# author: fanquanqing
#import datatime
from collect_eip_price import collect_eip_price
from collect_uhost_price import collect_uhost_price
from collect_sharebandwidth_price import collect_sharebandwidth_price
from collect_udisk_price import collect_udisk_price
from op_tools.api_influxdb import write_data_to_influxdb

def get_price(setting_info):
    '''
    各个实例price求和,并把实例信息发送到influxdb
    '''
    eip_price=0
    uhost_price=0
    sharebw_price=0
    udisk_price=0
    eip_price_info_list = collect_eip_price(setting_info)
    for eip_info in eip_price_info_list:
        eip_headers = eip_info.keys()
        eip_rows = []
        eip_rows.append(eip_info.values())
        #print eip_headers, eip_rows
        # 发送EIP数据到influxdb
        write_data_to_influxdb('EIP_info_daliy', eip_headers, eip_rows, ['IP', 'OperatorName', 'ChargeType'])
        eip_price += round(eip_info['Price'])

    uhost_price_info_list = collect_uhost_price(setting_info)
    for uhost_info in uhost_price_info_list:
        uhost_headers=uhost_info.keys()
        uhost_rows=[]
        uhost_rows.append(uhost_info.values())
        #print uhost_headers, uhost_rows
        # 发送云主机信息到influxdb
        write_data_to_influxdb('Uhost_info_daliy', uhost_headers, uhost_rows, ['Hostname','ChargeType'])
        uhost_price += round(uhost_info['Price'])

    sharebandwidth_info_list = collect_sharebandwidth_price(setting_info)
    for sharebandwidth_info in sharebandwidth_info_list:
        sharebw_headers=sharebandwidth_info.keys()
        sharebw_rows=[]
        sharebw_rows.append(sharebandwidth_info.values())
        #print sharebw_headers, sharebw_rows
        # 发送共享带宽信息到influxdb
        write_data_to_influxdb('ShareBandwidth_info_daliy',sharebw_headers,sharebw_rows,[])
        sharebw_price += round(sharebandwidth_info['Price'])

    udisk_info_list=collect_udisk_price(setting_info)
    for udisk_info in udisk_info_list:
        udisk_headers=udisk_info.keys()
        udisk_rows=[]
        udisk_rows.append(udisk_info.values())
        #print udisk_headers, udisk_rows
        write_data_to_influxdb('Udisk_info_daliy', udisk_headers,udisk_rows,['UHostName','Region'])
        udisk_price += round(udisk_info['Price'])
    # 托管机房价格
    physical_price = get_physical_host_price()
    total_price=eip_price+uhost_price+sharebw_price+udisk_price+physical_price
    # 价格列表
    price_list=[eip_price,uhost_price,sharebw_price,udisk_price,total_price]
    price_headers=['eip_price','uhost_price','sharebw_price','udisk_price','total_price']
    price_rows=[]
    price_rows.append(price_list)
    write_data_to_influxdb('Ucloud_price_total_daliy',price_headers,price_rows,[])

    #print uhost_price
    return uhost_price

# 托管机器价格(两个机柜一个10M外网)
def get_physical_host_price():
    host_price = 9000*2*12
    tg_cloud_switch_port_price = (288*2+217)*12
    tg_bandwidth_price = 10*90*12
    physical_price = host_price+tg_bandwidth_price+tg_cloud_switch_port_price
    return physical_price

if __name__ == '__main__':
    # 可用区列表
    #Regions_list = ['cn-bj2','hk']
    # 项目ID列表
    #Project_Id_list = ['org-shbbct','org-oddm1w']
    # 项目ID对应关系
    setting_info = {"chunyu":{"org-oddm1w":['cn-bj2','hk']}, "uhs":{"org-shbbct":["cn-bj2"]}}
    get_price(setting_info)

```

## grafana效果展示

> grafana跟influxDB配合的非常好，设置也非常简单。下面我在下面放几张效果图。

**简单查询语句**
![](http://or2jd66dq.bkt.clouddn.com/grafana_influxdb.png)

**table展示效果**
![](http://or2jd66dq.bkt.clouddn.com/grafana_ucloud_eip.png)

**价格图展示**
![](http://or2jd66dq.bkt.clouddn.com/grafana_ucloud_price.png)



## 获取实时带宽信息并发送报警到钉钉

> 这个跟上面的脚本没有关联，只是获取实时带宽使用量的报警脚本。
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from sdk import UcloudApiClient
from config import *
import sys
import json
import urllib2
import time
#from op_tools import falcon

#实例化 API 句柄
localtime = time.asctime( time.localtime(time.time()) )

def get_eip_info():
    '''
    获取EIP对应的主机名，以及EIP信息返回
    '''
    arg_length = len(sys.argv)
    ApiClient = UcloudApiClient(base_url, public_key, private_key)
    Parameters={"Action":"DescribeEIP", "Region":"cn-bj2"}
    response = ApiClient.get("/", Parameters );
    eip_info_list = response['EIPSet']
    eip_dic = {}
    for eip_info in eip_info_list:
        # EIP绑定主机名
        eip_host = eip_info['Resource']['ResourceName']
        eip_ip = eip_info['EIPAddr'][0]['IP'].encode('utf-8')
        eip_id = eip_info['EIPId']
        dic = {}
        dic[eip_ip] = eip_id
        #print "eip_host:%s,dic:%s" % (eip_host,dic)
        eip_dic[eip_host] = dic
    #print len(eip_dic)
    eip_dic['nginx-online1'] = {'106.75.28.177': u'eip-00gv0l'}
    return eip_dic

def get_eip_usage(eip_dic):
    '''
    获取每个EIP的实时用量(要求EIPid),
    return list:[{ip:usage}...]
    '''
    eip_usage_dic = {}
    for eip_host in eip_dic:
        #eip_useage_dic = {}
        eip_id=eip_dic[eip_host].values()[0].encode("utf-8")
        #print eip_id
        ApiClient = UcloudApiClient(base_url, public_key, private_key)
        Parameters={
                    "Action":"DescribeBandwidthUsage",
                    "Region":"cn-bj2",
                    "EIPIds.1":eip_id,
                }
        response = ApiClient.get("/", Parameters);
        #print json.dumps(response, sort_keys=True, indent=4, separators=(',', ': '))
        #print response
        eip_usage = response['EIPSet'][0]['CurBandwidth']
        eip_usage_dic[eip_host]=eip_usage
        #eip_usage_list.append(eip_useage_dic)
    return eip_usage_dic

def sendto_falcon(eip_usage_list):
    '''
    报警发送到falcon
    '''
    collect_step = 60
    counter_type = falcon.CounterType.GAUGE
    metric = "bandwidthusage"
    for eip_usage in eip_usage_list:
        tags="host=" + eip_usage.keys()[0]
        value=eip_usage.values()[0]
        #print value

def get_sharebw_info():
    '''
    获取共享带宽的带宽大小，以及所包含的EIP,
    return list:[{'eiplist':[ip1,ip1],'bandwidth':20}...]
    '''

    ApiClient = UcloudApiClient(base_url, public_key, private_key)
    Parameters={"Action":"DescribeShareBandwidth", "Region":"cn-bj2"}
    response = ApiClient.get("/", Parameters );
    #print json.dumps(response, sort_keys=True, indent=4, separators=(',', ': '))
    share_bw_list = response['DataSet']
    share_bw_info = []
    for share_bw in share_bw_list:
        share_bw_dic = {}
        bandwidth = share_bw['ShareBandwidth']
        eip_list = []
        for eip_dic in share_bw['EIPSet']:
            ip = eip_dic['EIPAddr'][0]['IP']
            eip_list.append(ip)
        share_bw_dic['bandwidth']=bandwidth
        share_bw_dic['eiplist']=eip_list
        share_bw_info.append(share_bw_dic)
    return share_bw_info

def sum():
    '''
    返回比值，并简单记录log到/var/log/ubandwidth.log
    '''
    eip_dic = get_eip_info()
    #print eip_dic
    eip_to_host = {}
    # 通过eip_dic获取IP-host对此应关系dic
    for host in eip_dic:
        eip = eip_dic[host].keys()[0]
        eip_to_host[eip]=host
    #print eip_to_host
    eip_usage_dic = get_eip_usage(eip_dic)
    #print eip_usage_dic
    share_bw_info = get_sharebw_info()
    #print share_bw_info
    #各带宽和与带宽比值
    ratios = {}
    for share_bandwidth in share_bw_info:
        bandwidth = share_bandwidth['bandwidth']
        sum = 0
        for eip in share_bandwidth['eiplist']:
            host = eip_to_host[eip]
            usage = eip_usage_dic[host]
            sum += usage
        #带宽用量与带宽的比值
        ratio = sum/bandwidth
        parts = [str(bandwidth),str(sum),str(ratio)]
        log =localtime + ' ' + ','.join(parts) + '\n'
        with open('/var/log/ubandwidth.log','a') as f:
            f.write(log)
            f.close()

        ratios[bandwidth]=ratio

    return ratios


def send_to_dingtalk(content):

    url = "https://oapi.dingtalk.com/robot/send?access_token=17cf865229a63452ff411243b53d64949d5a54b1ee8774e20e1ec7d4c5d60f43"
    #con={"msgtype":"text","text":{"content":content},"isAtAll": "true"}
    con={"msgtype":"markdown","markdown":{"title":"ucloud共享带宽报警","text":content},"isAtAll": "ture"}
    jd=json.dumps(con)
    req=urllib2.Request(url,jd)
    req.add_header('Content-Type', 'application/json')
    response=urllib2.urlopen(req)

if __name__ == '__main__':
    ratios=sum()
    localtime = time.asctime( time.localtime(time.time()) )
    for bw in ratios:
        if ratios[bw] > 0.8:
            content = u"# **ucloud共享带宽报警** - %d兆那个。。。\n\n - 用量超过百分之80 \n - **值**:%f \n > [请排查...](https://console.ucloud.cn/unet/sharebandwidth)" % (bw,ratios[bw])
            send_to_dingtalk(content)
```




