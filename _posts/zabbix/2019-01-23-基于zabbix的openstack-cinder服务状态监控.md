---
layout: post
title: 基于zabbix的openstack-cinder服务状态监控
category: 监控
tags: [openstack, zabbix, 服务监控]
description: 前面我们讨论了如何使用zabbix自动发现监控openstack nova服务状态，本文使用类似的方法监控openstack cinder服务状态。
---


# 基于zabbix的openstack-cinder服务状态监控

前面我们讨论了如何使用zabbix自动发现监控openstack nova服务状态，本文使用类似的方法监控openstack cinder服务状态。

## 1 使用控制台获取相关服务状态

使用如下命令可以获取当前openstack的nova服务状态
```
cinder service-list

+------------------+-------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |     Host    | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+-------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler |    cinder   | nova | enabled |   up  | 2019-01-26T02:44:32.000000 |        -        |
|  cinder-volume   | cinder@ceph | nova | enabled |   up  | 2019-01-26T02:44:37.000000 |        -        |
+------------------+-------------+------+---------+-------+----------------------------+-----------------+
```

可以看到，正常情况下，所有的状态均应为 enabled 和 up，我们也是监控所有服务的status和state。

## 2 使用zabbix自动发现获取cinder服务信息

### 2.1 zabbix自动发现配置与Item
zabbix自动发现配置
```
openstack.cinder.discovery
```
zabbix自动发现Item
```
openstack.cinder.status[state, {#BINARY}, {#HOST}]
openstack.cinder.status[status, {#BINARY}, {#HOST}]
```
同时设置对应的Trigger,当节点服务异常时就可以及时发出告警了。

考虑到便于grafana的展示，所以也做了状态的合并。设置如下固定Item
```
openstack.cinder.all_status[state]
openstack.cinder.all_status[status]
```

### 2.2 zabbix-agent配置文件
```
UserParameter=openstack.cinder.discovery,python /etc/zabbix/zabbix_agentd.d/openstack-cinder.py --item discovery

UserParameter=openstack.cinder.status[*],python /etc/zabbix/zabbix_agentd.d/openstack-cinder.py --item $1 --binary $2 --host $3

UserParameter=openstack.cinder.all_status[*],python /etc/zabbix/zabbix_agentd.d/openstack-cinder.py --item all_status --item1 $1
```

### 2.3 zabbix相关监控与查询脚本
/etc/zabbix/zabbix_agentd.d/openstack-cinder.py
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import json
from optparse import OptionParser
from cinderclient import client as client


#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'

cinder_client = client.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    options = parse_args()
    if options.item=="discovery":
        service_list()
    elif options.item=="all_status":
        service_status(options)
    else:
        service_moniter(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "status", "state", "disabled_reason", "all_status"]
    parser.add_option("", "--item", dest="item", help="", action="store", type="string", default=None)
    parser.add_option("", "--binary", dest="binary", help="", action="store", type="string", default=None)
    parser.add_option("", "--host", dest="host", help="", action="store", type="string", default=None)
    parser.add_option("", "--item1", dest="item1", help="", action="store", type="string", default=None)
    (options, args) = parser.parse_args()
    if options.item not in valid_item:
        parser.error("Item has to be one of: "+", ".join(valid_item))
    return options

#获取服务列表
def service_list():
    r = {"data":[]}
    services = cinder_client.services.list()
    for service in services:
        service_info = service._info.copy()
        r['data'].append( {"{#BINARY}":service_info["binary"], "{#HOST}":service_info["host"], "{#ZONE}":service_info["zone"]} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))


#获取所有节点状态的合成
def service_status(options):
    services = cinder_client.services.list()
    status = 0
    for service in services:
        service_info = service._info.copy()
        if service_info[options.item1]=="down" or service_info[options.item1]=="disabled":
            status = 1
            break
        else:
            pass
    print(status)

#获取对应服务的监控信息
def service_moniter(options):
    services = cinder_client.services.list(binary = options.binary,host = options.host)
    for service in services:
        service_info = service._info.copy()
        if service_info[options.item]=="up" or service_info[options.item]=="enabled":
            print 0
        elif service_info[options.item]=="down" or service_info[options.item]=="disabled":
            print 1
        else:
            print -1

if __name__ == "__main__":
    main()
```

# 参考资料
1. nova 命令汇总四 ——计算相关命令,http://blog.51cto.com/13788458/2129157
2. The novaclient Python API,https://docs.openstack.org/python-novaclient/latest/reference/api/index.html
3. GitHub - larsks/openstack-api-samples,https://github.com/larsks/openstack-api-samples

# 如有疑问，欢迎交流
