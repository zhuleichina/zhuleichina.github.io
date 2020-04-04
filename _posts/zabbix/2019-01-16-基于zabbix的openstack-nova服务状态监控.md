---
layout: post
title: 基于zabbix的openstack-nova服务状态监控
category: 监控
tags: [openstack, zabbix, 服务监控]
description: 对于Openstack运维人员来说，需要关注openstack中，计算节点的服务状态，当发现服务异常时，应当及时处理。
---


# 基于zabbix的openstack-nova服务状态监控

对于Openstack运维人员来说，需要关注openstack中，计算节点的服务状态，当发现服务异常时，应当及时处理。

本文讨论基于nova python-api的服务状态监控

## 1 使用控制台获取相关服务状态

使用如下命令可以获取当前openstack的nova服务状态
```
nova service-list

+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host         | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-monitor     | controler02  | internal | enabled | up    | 2019-01-16T07:34:10.000000 | -               |
| 2  | nova-monitor     | controller01 | internal | enabled | up    | 2019-01-16T07:34:09.000000 | -               |
| 3  | nova-consoleauth | controler02  | internal | enabled | down  | 2017-10-09T09:46:37.000000 | -               |
| 4  | nova-consoleauth | controller01 | internal | enabled | up    | 2019-01-16T07:34:10.000000 | -               |
| 5  | nova-scheduler   | controler02  | internal | enabled | down  | 2017-10-09T09:46:37.000000 | -               |
| 6  | nova-scheduler   | controller01 | internal | enabled | up    | 2019-01-16T07:34:05.000000 | -               |
| 7  | nova-conductor   | controler02  | internal | enabled | down  | 2017-10-09T09:46:27.000000 | -               |
| 10 | nova-conductor   | controller01 | internal | enabled | up    | 2019-01-16T07:34:10.000000 | -               |
| 14 | nova-cert        | controler02  | internal | enabled | down  | 2017-10-09T09:46:37.000000 | -               |
| 15 | nova-cert        | controller01 | internal | enabled | up    | 2019-01-16T07:34:06.000000 | -               |
| 17 | nova-storage     | computer05   | internal | enabled | up    | 2019-01-16T07:34:12.000000 | -               |
| 19 | nova-compute     | computer04   | IMS      | enabled | up    | 2019-01-16T07:34:06.000000 | -               |
| 21 | nova-compute     | computer03   | IMS      | enabled | up    | 2019-01-16T07:34:08.000000 | -               |
| 23 | nova-compute     | computer05   | paas     | enabled | up    | 2019-01-16T07:34:06.000000 | -               |
| 25 | nova-compute     | computer06   | paas     | enabled | up    | 2019-01-16T07:34:04.000000 | -               |
| 27 | nova-compute     | computer07   | paas     | enabled | up    | 2019-01-16T07:34:03.000000 | -               |
| 29 | nova-storage     | computer06   | internal | enabled | up    | 2019-01-16T07:34:05.000000 | -               |
| 31 | nova-compute     | computer08   | IMS      | enabled | up    | 2019-01-16T07:34:08.000000 | -               |
| 32 | nova-storage     | computer04   | internal | enabled | up    | 2019-01-16T07:34:07.000000 | -               |
| 34 | nova-storage     | computer07   | internal | enabled | up    | 2019-01-16T07:34:06.000000 | -               |
| 36 | nova-storage     | computer03   | internal | enabled | up    | 2019-01-16T07:34:05.000000 | -               |
| 38 | nova-storage     | computer08   | internal | enabled | up    | 2019-01-16T07:34:02.000000 | -               |
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
```

可以看到，除了controler01和controler02因为主备的问题而有down的情况，其他节点应该均为enable和up的状态，我们监控的也就是这个状态。

## 2 使用zabbix自动发现获取相关服务信息

### 2.1 zabbix自动发现配置与Item
zabbix自动发现配置
```
openstack.service.discovery
```
zabbix自动发现Item
```
openstack.service.status[state, {#BINARY}, {#HOST}]
openstack.service.status[status, {#BINARY}, {#HOST}]
```
同时设置对应的Trigger, 当节点服务异常时就可以及时发出告警了。


### 2.2 zabbix-agent配置文件
```
UserParameter=openstack.service.discovery,python /etc/zabbix/zabbix_agentd.d/openstack-service.py --item discovery

UserParameter=openstack.service.status[*],python /etc/zabbix/zabbix_agentd.d/openstack-service.py --item $1 --binary $2 --host $3
```

### 2.3 zabbix相关监控与查询脚本
/etc/zabbix/zabbix_agentd.d/openstack-service.py
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import json
from optparse import OptionParser
from novaclient import client as noclient
from novaclient import utils


#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'

nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    options = parse_args()
    if options.item=="discovery":
        service_list()
    else:
        service_moniter(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "status", "state", "disabled_reason"]
    parser.add_option("", "--item", dest="item", help="", action="store", type="string", default=None)
    parser.add_option("", "--binary", dest="binary", help="", action="store", type="string", default=None)
    parser.add_option("", "--host", dest="host", help="", action="store", type="string", default=None)
    (options, args) = parser.parse_args()
    if options.item not in valid_item:
        parser.error("Item has to be one of: "+", ".join(valid_item))
    return options

#获取服务列表
def service_list():
    r = {"data":[]}
    services = nova_client.services.list()
    for service in services:
        service_info = service._info.copy()
        #排除两个控制节点
        if service_info["host"] == "controller01" or service_info["host"] == "controller02":
            pass
        else:
            r['data'].append( {"{#BINARY}":service_info["binary"], "{#HOST}":service_info["host"], "{#ZONE}":service_info["zone"]} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取对应服务的监控信息
def service_moniter(options):
    services = nova_client.services.list(host = options.host, binary = options.binary)
    for service in services:
        service_info = service._info.copy()
        print (service_info[options.item])

if __name__ == "__main__":
    main()
```

## 3 配合grafana展示

使用此方法，在控制节点就可以及时监控相关服务的状态，同时由于采用了Openstack控制台相同的数据，结果上也更加准确。同时计算节点变动后，不需要修改任何监控参数，即可自动发现与调整。

grafana进行展示时，结果只能配置为数字，另外因为自动发现的服务数量较多，且经常因为缩容，扩容而变动，因此想到可以在grafana只展示所有服务的合成状态，专门供grafana展示使用。

zabbix增加监控项
```
openstack.service.all_status[state]
openstack.service.all_status[status]
```

zabbix修改后配置文件
```
UserParameter=openstack.service.discovery,python /etc/zabbix/zabbix_agentd.d/openstack-service.py --item discovery

UserParameter=openstack.service.status[*],python /etc/zabbix/zabbix_agentd.d/openstack-service.py --item $1 --binary $2 --host $3

UserParameter=openstack.service.all_status[*],python /etc/zabbix/zabbix_agentd.d/openstack-service.py --item all_status --item1 $1
```

修改后脚本
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import json
from optparse import OptionParser
from novaclient import client as noclient
from novaclient import utils


#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'

nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

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
    services = nova_client.services.list()
    for service in services:
        service_info = service._info.copy()
        if service_info["host"] == "Controller02" or service_info["host"] == "Controller01":
            pass
        else:
            r['data'].append( {"{#BINARY}":service_info["binary"], "{#HOST}":service_info["host"], "{#ZONE}":service_info["zone"]} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取所有节点状态的合成
def service_status(options):
    services = nova_client.services.list()
    status = 0
    for service in services:
        service_info = service._info.copy()
        if service_info["host"] == "Controller02" or service_info["host"] == "Controller01":
            pass
        else:
            if service_info[options.item1]=="down" or service_info[options.item1]=="disabled":
                status = 1
                break
            else:
                pass
    print(status)

#获取对应服务的监控信息
def service_moniter(options):
    services = nova_client.services.list(host = options.host, binary = options.binary)
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
