---
layout: post
title: 基于ceilometer的openstack虚机资源监控
category: 工具
tags: [ceilometer, openstack, zabbix, 资源健康]
description: 本文利用ceilometer自带的监控项，结合zabbix做数据提取与展示，可以作为openstack虚机资源监控的一种尝试。
---

# 基于ceilometer的openstack虚机资源监控

对于Openstack运维人员来说，掌握Openstack云平台内虚机运行情况有助于快速定位问题，但现有的Openstack自带监控界面显示等均不易使用与扩展。

本文利用ceilometer自带的监控项，结合zabbix做数据提取与展示，可以作为openstack虚机资源监控的一种尝试。

## 1 Openstack虚机ceilometer监控项获取

使用如下命令可以获取当前ceilometer的所有监控项
```
ceilometer meter-list
```
该命令获取所有的监控项所以查看起来不易，我们可以获取虚机ID后，筛选出其中一台虚机作为展示

```
#获取虚机ID列表
nova list --all

------------------------------------------------+
| ID                                   | Name              | Tenant ID                        | Status | Task State | Power State | Networks                                                                        |
+--------------------------------------+-------------------+----------------------------------+--------+------------+-------------+---------------------------------------------------------------------------------+
| 823bf8b4-96b4-4614-ab0e-49fba80bd13d | Kafka_1           | d448a43772e5434592baf9217e9a1b82 | ACTIVE | -          | Running     | dap_admin=172.31.0.3                                                            |
| caee156f-166a-45b7-af0a-696890021cd7 | Kafka_2           | d448a43772e5434592baf9217e9a1b82 | ACTIVE | -          | Running     | dap_admin=172.31.0.4                                                            |
| 1745501e-26b5-4d89-9a60-9da087f0476f | Kafka_3           | d448a43772e5434592baf9217e9a1b82 | ACTIVE | -          | Running     | dap_admin=172.31.0.5                                                            |
| c4a519b3-8faf-4c38-9489-06a551b09b07 | Redis             | d448a43772e5434592baf9217e9a1b82 | ACTIVE | -          | Running     | dap_admin=172.31.0.6 

```

筛选出其中一台虚机的监控项
```
ceilometer meter-list | grep 823bf8b4-96b4-4614-ab0e-49fba80bd13d

+---------------------------------------------+------------+-----------+-----------------------------------------------------------------------+----------------------------------+----------------------------------+
| Name                                        | Type       | Unit      | Resource ID                                                           | User ID                          | Project ID                       |
+---------------------------------------------+------------+-----------+-----------------------------------------------------------------------+----------------------------------+----------------------------------+


| cpu                                         | cumulative | ns        | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| cpu_util                                    | gauge      | %         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.allocation                             | gauge      | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.capacity                               | gauge      | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.allocation                      | gauge      | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.capacity                        | gauge      | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.read.bytes                      | cumulative | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.read.bytes.rate                 | gauge      | B/s       | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.usage                           | gauge      | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.write.bytes                     | cumulative | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.device.write.bytes.rate                | gauge      | B/s       | 823bf8b4-96b4-4614-ab0e-49fba80bd13d-vda                              | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.read.bytes                             | cumulative | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.read.bytes.rate                        | gauge      | B/s       | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.read.requests                          | cumulative | request   | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.read.requests.rate                     | gauge      | request/s | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.total.size                             | gauge      | GB        | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.usage                                  | gauge      | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.write.bytes                            | cumulative | B         | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.write.bytes.rate                       | gauge      | B/s       | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.write.requests                         | cumulative | request   | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| disk.write.requests.rate                    | gauge      | request/s | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| instance                                    | gauge      | instance  | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| memory                                      | gauge      | MB        | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| memory.usage                                | gauge      | MB        | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.incoming.bytes                      | cumulative | B         | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.incoming.bytes.rate                 | gauge      | B/s       | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.incoming.packets                    | cumulative | packet    | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.incoming.packets.drop               | cumulative | packet    | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.incoming.packets.error              | cumulative | packet    | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.incoming.packets.rate               | gauge      | packet/s  | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.outgoing.bytes                      | cumulative | B         | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.outgoing.bytes.rate                 | gauge      | B/s       | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.outgoing.packets                    | cumulative | packet    | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.outgoing.packets.drop               | cumulative | packet    | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.outgoing.packets.error              | cumulative | packet    | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| network.outgoing.packets.rate               | gauge      | packet/s  | instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| poweron                                     | gauge      | N/A       | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
| vcpus                                       | gauge      | vcpu      | 823bf8b4-96b4-4614-ab0e-49fba80bd13d                                  | 6e2f1fdf1e3c4cae95ce8bb09ec99431 | d448a43772e5434592baf9217e9a1b82 |
```

对于某个监控项来说，如果我们知道其resource_id和监控项就可以使用ceilometer就行查询，如：
```
#查询基础信息cpu使用率
# -m为监控项,-q为查询条件，-l为查询数目，这里是显示最近一条

ceilometer sample-list -m cpu_util -q'resource_id=823bf8b4-96b4-4614-ab0e-49fba80bd13d' -l 1

+--------------------------------------+----------+-------+----------------+------+---------------------+
| Resource ID                          | Name     | Type  | Volume         | Unit | Timestamp           |
+--------------------------------------+----------+-------+----------------+------+---------------------+
| 823bf8b4-96b4-4614-ab0e-49fba80bd13d | cpu_util | gauge | 0.162583056478 | %    | 2018-07-04T09:43:28 |
+--------------------------------------+----------+-------+----------------+------+---------------------+


#查询网络监控某个网口入口流量速度

ceilometer sample-list -m network.incoming.bytes.rate -q'resource_id=instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce' -l 1

+-----------------------------------------------------------------------+-----------------------------+-------+---------------+------+---------------------+
| Resource ID                                                           | Name                        | Type  | Volume        | Unit | Timestamp           |
+-----------------------------------------------------------------------+-----------------------------+-------+---------------+------+---------------------+
| instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce | network.incoming.bytes.rate | gauge | 92.7666666667 | B/s  | 2018-07-04T09:48:28 |
+-----------------------------------------------------------------------+-----------------------------+-------+---------------+------+---------------------+

```

下面我们分开讨论虚机基础信息与网络信息的获取

## 2 VM虚机基础信息监控

### 2.1 获取虚机ID列表

对于命令行来说，我们可以使用如下命令获取所有虚机列表：
```
nova list --all
```

但对于zabbix来说，使用命令行结果提取会不是很方便，这里我们考虑使用novaclient python-api 来获取，代码如下：
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

import json

from novaclient import client as noclient


#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'

#creating an authenticated client
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def vm_list():
    r = {"data":[]}

    novas = nova_client.servers.list(detailed='detailed', search_opts={'all_tenants': 1})
    for nova in novas:
        r['data'].append( {"{#VMNAME}":nova.name, "{#VMID}":nova.id} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

if __name__ == "__main__":
    vm_list()

```

### 2.2 zabbix自动发现LLD配置

获取虚机ID后就可以根据监控项获取对应的监控内容了，zabbix自动发现LLD设置如下：

```
#自动发现key:

openstack.vm.discovery
```

监控Item,key设置如下：
```
openstack.ceilometer[cpu,{#VMID}]
openstack.ceilometer[cpu_util,{#VMID}]
openstack.ceilometer[disk.allocation,{#VMID}]openstack.ceilometer[disk.capacity,{#VMID}]
openstack.ceilometer[disk.read.bytes,{#VMID}]
openstack.ceilometer[disk.read.bytes.rate,{#VMID}]
openstack.ceilometer[disk.read.requests,{#VMID}]
openstack.ceilometer[disk.read.requests.rate,{#VMID}]
openstack.ceilometer[disk.total.size,{#VMID}]
openstack.ceilometer[disk.usage,{#VMID}]
openstack.ceilometer[disk.write.bytes,{#VMID}]
openstack.ceilometer[disk.write.bytes.rate,{#VMID}]
openstack.ceilometer[disk.write.requests,{#VMID}]
openstack.ceilometer[disk.write.requests.rate,{#VMID}]
openstack.ceilometer[instance,{#VMID}]
openstack.ceilometer[memory,{#VMID}]
openstack.ceilometer[memory.usage,{#VMID}]
openstack.ceilometer[poweron,{#VMID}]
openstack.ceilometer[vcpus,{#VMID}]
```
其中{#VMID}为程序获取的虚机ID，第一个参数是虚机的基础信息参数。

### 2.3 zabbix-agent配置与对应的脚本

对应的zabbix-agent.conf配置如下：

```
# /etc/zabbix/zabbix_agentd.d/userparameter_openstack-vm.conf

UserParameter=openstack.vm.discovery,/etc/zabbix/zabbix_agentd.d/openstack-vm.py --item discovery

UserParameter=openstack.ceilometer[*],/etc/zabbix/zabbix_agentd.d/openstack-vm.py --item $1 --uuid $2
```

对应的监控脚本获取监控信息如下：
```
# /etc/zabbix/zabbix_agentd.d/openstack-vm.py

#!/usr/bin/python
# -*- coding: utf-8 -*-

import json
from optparse import OptionParser

from ceilometerclient import client as cmclient
from novaclient import client as noclient


#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'

#creating an authenticated client
ceilometer_client = cmclient.get_client(2,**keystone)
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    options = parse_args()
    if options.item=="discovery":
        vm_list()
    elif options.item=="net_discovery":
        net_list()
    else:
        ceilometer_query(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "cpu", "cpu_util", "disk.allocation", "disk.capacity", "disk.read.bytes", "disk.read.bytes.rate", 
    "disk.read.requests", "disk.read.requests.rate", "disk.total.size", "disk.usage", "disk.write.bytes", "disk.write.bytes.rate", 
    "disk.write.requests", "disk.write.requests.rate", "instance", "memory", "memory.usage", "poweron", "vcpus"]
    parser.add_option("", "--item", dest="item", help="", action="store", type="string", default=None)
    parser.add_option("", "--uuid", dest="uuid", help="", action="store", type="string", default=None)
    (options, args) = parser.parse_args()
    if options.item not in valid_item:
        parser.error("Item has to be one of: "+", ".join(valid_item))
    return options
  
#使用nova api获取虚机列表
def vm_list():
    r = {"data":[]}

    novas = nova_client.servers.list(detailed='detailed', search_opts={'all_tenants': 1})
    for nova in novas:
        r['data'].append( {"{#VMNAME}":nova.name, "{#VMID}":nova.id} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取对应监控项的监控值
def ceilometer_query(options):
    fields = {'meter_name': options.item,
              'q': [{"field": "resource_id", "op": "eq", "value": options.uuid}],
              'limit': 1}
    samples = ceilometer_client.samples.list(**fields)

    print samples[0].counter_volume

if __name__ == "__main__":
    main()

```

在zabbix上进行对应的配置后重启，将模版应用于主机，此时应当可以发现所有的虚机信息，并能够返回ceilometer的测量值。

## 3 VM虚机网络信息监控

对于网络信息来说，引起resource_id不是直接的虚机ID，而是经过组合后的信息，我们将先对其进行解析：

```
instance-00000067-823bf8b4-96b4-4614-ab0e-49fba80bd13d-ovkb478c1ea-ce

#instance-00000067 
为虚机的instance号，命令行可以通过 nova show 823bf8b4-96b4-4614-ab0e-49fba80bd13d 获取

#823bf8b4-96b4-4614-ab0e-49fba80bd13d 
是虚机ID

#ovkb478c1ea-ce
其中b478c1ea-ce为虚机的port信息，命令行可以通过nova interface-list 823bf8b4-96b4-4614-ab0e-49fba80bd13d 获取

通过查询对应的novaclient源码可以指导命令行是如何打印出相关信息的，我们特可以通过对应的 python-api 进行获取
```

### 3.1 获取网络信息resource_id获取

代码如下：
```
def net_list():
    r = {"data":[]}
    novas = nova_client.servers.list(detailed='detailed', search_opts={'all_tenants': 1})
    for nova in novas:
        nova_info = nova._info.copy()
        nets = nova.interface_list()
        for net in nets:
            net_info = net._info.copy()
            resource_id = nova_info["OS-EXT-SRV-ATTR:instance_name"] + "-" + nova_info["id"] + "-"  + "ovk"  + net_info["port_id"][0:11]
            if net_info["fixed_ips"]:
                ip_address = net_info["fixed_ips"][0]["ip_address"]
                r['data'].append( {"{#VMNAME}":nova.name, "{#NETID}":resource_id, "{#IPADDRESS}":ip_address} )
            else:
                r['data'].append( {"{#VMNAME}":nova.name, "{#NETID}":resource_id, "{#IPADDRESS}":"no ip"} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

```

### 3.2 zabbix自动发现LLD配置

获取网络 resource_id 列表后就可以进行对应的zabbix配置了，zabbix-LLD配置如下：
```
#自动发现key

openstack.vm.net_discovery
```
对应的监控Item,key配置如下：
```
openstack.ceilometer[network.incoming.bytes,{#NETID}]
openstack.ceilometer[network.incoming.bytes.rate,{#NETID}]
openstack.ceilometer[network.incoming.packets,{#NETID}]
openstack.ceilometer[network.incoming.packets.drop,{#NETID}]
openstack.ceilometer[network.incoming.packets.error,{#NETID}]
openstack.ceilometer[network.incoming.packets.rate,{#NETID}]
openstack.ceilometer[network.outgoing.bytes,{#NETID}]
openstack.ceilometer[network.outgoing.bytes.rate,{#NETID}]
openstack.ceilometer[network.outgoing.packets,{#NETID}]
openstack.ceilometer[network.outgoing.packets.drop,{#NETID}]
openstack.ceilometer[network.outgoing.packets.error,{#NETID}]
openstack.ceilometer[network.outgoing.packets.rate,{#NETID}]
```
### 3.3 zabbix-agent配置与对应的脚本

对应的zabbix-agent.conf配置如下(包含基础信息监控)：

```
# /etc/zabbix/zabbix_agentd.d/userparameter_openstack-vm.conf

UserParameter=openstack.vm.discovery,/etc/zabbix/zabbix_agentd.d/openstack-vm.py --item discovery

UserParameter=openstack.vm.net_discovery,/etc/zabbix/zabbix_agentd.d/openstack-vm.py --item net_discovery

UserParameter=openstack.ceilometer[*],/etc/zabbix/zabbix_agentd.d/openstack-vm.py --item $1 --uuid $2
```

对应的监控脚本获取监控信息如下（包含基础信息监控）：
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

import json
from optparse import OptionParser

from ceilometerclient import client as cmclient
from novaclient import client as noclient


#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'

#creating an authenticated client
ceilometer_client = cmclient.get_client(2,**keystone)
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    options = parse_args()
    if options.item=="discovery":
        vm_list()
    elif options.item=="net_discovery":
        net_list()
    else:
        ceilometer_query(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "cpu", "net_discovery", "cpu_util", "disk.allocation", "disk.capacity", "disk.read.bytes", "disk.read.bytes.rate", 
    "disk.read.requests", "disk.read.requests.rate", "disk.total.size", "disk.usage", "disk.write.bytes", "disk.write.bytes.rate", 
    "disk.write.requests", "disk.write.requests.rate", "instance", "memory", "memory.usage", "poweron", "vcpus", 
    "network.incoming.bytes", "network.incoming.bytes.rate", "network.outgoing.bytes", "network.outgoing.bytes.rate", 
    "network.incoming.packets", "network.incoming.packets.rate", "network.outgoing.packets", "network.outgoing.packets.rate", 
    "network.incoming.packets.drop", "network.incoming.packets.error", "network.outgoing.packets.drop", "network.outgoing.packets.error"]
    parser.add_option("", "--item", dest="item", help="", action="store", type="string", default=None)
    parser.add_option("", "--uuid", dest="uuid", help="", action="store", type="string", default=None)
    (options, args) = parser.parse_args()
    if options.item not in valid_item:
        parser.error("Item has to be one of: "+", ".join(valid_item))
    return options
  
#使用nova api获取虚机列表
def vm_list():
    r = {"data":[]}

    novas = nova_client.servers.list(detailed='detailed', search_opts={'all_tenants': 1})
    for nova in novas:
        r['data'].append( {"{#VMNAME}":nova.name, "{#VMID}":nova.id} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#使用nova api获取虚机网络，组合为网卡ceilometer查询ID
def net_list():
    r = {"data":[]}
    novas = nova_client.servers.list(detailed='detailed', search_opts={'all_tenants': 1})
    for nova in novas:
        nova_info = nova._info.copy()
        nets = nova.interface_list()
        for net in nets:
            net_info = net._info.copy()
            resource_id = nova_info["OS-EXT-SRV-ATTR:instance_name"] + "-" + nova_info["id"] + "-"  + "ovk"  + net_info["port_id"][0:11]
            if net_info["fixed_ips"]:
                ip_address = net_info["fixed_ips"][0]["ip_address"]
                r['data'].append( {"{#VMNAME}":nova.name, "{#NETID}":resource_id, "{#IPADDRESS}":ip_address} )
            else:
                r['data'].append( {"{#VMNAME}":nova.name, "{#NETID}":resource_id, "{#IPADDRESS}":"no ip"} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取对应监控项的监控值
def ceilometer_query(options):
    fields = {'meter_name': options.item,
              'q': [{"field": "resource_id", "op": "eq", "value": options.uuid}],
              'limit': 1}
    samples = ceilometer_client.samples.list(**fields)

    print samples[0].counter_volume

if __name__ == "__main__":
    main()

```

在zabbix上进行对应的配置后重启，将模版应用于主机，此时应当可以发现所有的虚机网络信息，并能够返回ceilometer的测量值。

## 4 遇到的部分问题
### 4.1 net_list函数超时问题

因脚本中 net_list()函数执行时间稍长，10s左右，默认的超时时间会得不到结果，zabbix会报错Timeout Error，需要增加zabbix-agent,和zabbix-server如下配置：
```
/etc/zabbix/zabbix_agentd.conf
/etc/zabbix/zabbix_server.conf

Timeout=30
```

### 4.2整体性能问题

因当虚机数目较多时，无论是net_list()函数执行时间，还是zabbix的监控数目都呈线性增长，会存在一定的性能瓶颈，需要对脚本进行优化或者对部分监控项做裁剪，目前还没有找到好的解决办法。

# 参考资料
1. Ceilometer详解,https://blog.csdn.net/u010305706/article/details/51001622
2. zabbix监控openstack的资源使用情况,http://blog.51cto.com/superbigsea/1856993
3. The novaclient Python API,https://docs.openstack.org/python-novaclient/latest/reference/api/index.html
4. GitHub - larsks/openstack-api-samples,https://github.com/larsks/openstack-api-samples

# 如有疑问，欢迎交流
