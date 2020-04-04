---
layout: post
title: 基于zabbix的openstack系统资源监控
category: 监控
tags: [openstack, zabbix, 系统资源]
description: 本文将介绍使用zabbix结合openstack python-api对openstack系统资源进行监控。
---

# 基于zabbix的openstack系统资源监控

对于Openstack运维人员来说，需要掌握Openstack云平台系统资源的整体运行情况，包括域(AZ)的cpu/menmory等资源使用情况；本文将介绍使用zabbix结合openstack python-api对openstack系统资源进行监控。

## 1 使用控制台获取域(AZ)监控信息

使用如下命令可以获取当前openstack的所有可用域
```
nova availability-zone-list

+-----------------------+----------------------------------------+
| Name                  | Status                                 |
+-----------------------+----------------------------------------+
| internal              | available                              |
| |- computer03         |                                        |
| | |- nova-storage     | enabled :-) 2018-10-18T10:56:07.000000 |
| |- computer04         |                                        |
| | |- nova-storage     | enabled :-) 2018-10-18T10:56:08.000000 |
| |- computer05         |                                        |
| | |- nova-storage     | enabled :-) 2018-10-18T10:56:13.000000 |
| |- computer06         |                                        |
| | |- nova-storage     | enabled :-) 2018-10-18T10:56:05.000000 |
| |- computer07         |                                        |
| | |- nova-storage     | enabled :-) 2018-10-18T10:56:05.000000 |
| |- computer08         |                                        |
| | |- nova-storage     | enabled :-) 2018-10-18T10:56:07.000000 |
| |- controler02        |                                        |
| | |- nova-conductor   | enabled XXX 2017-10-09T09:46:27.000000 |
| | |- nova-consoleauth | enabled XXX 2017-10-09T09:46:37.000000 |
| | |- nova-monitor     | enabled :-) 2018-10-18T10:56:11.000000 |
| | |- nova-scheduler   | enabled XXX 2017-10-09T09:46:37.000000 |
| | |- nova-cert        | enabled XXX 2017-10-09T09:46:37.000000 |
| |- controller01       |                                        |
| | |- nova-conductor   | enabled :-) 2018-10-18T10:56:11.000000 |
| | |- nova-consoleauth | enabled :-) 2018-10-18T10:56:04.000000 |
| | |- nova-monitor     | enabled :-) 2018-10-18T10:56:07.000000 |
| | |- nova-scheduler   | enabled :-) 2018-10-18T10:56:09.000000 |
| | |- nova-cert        | enabled :-) 2018-10-18T10:56:12.000000 |
| IMS                   | available                              |
| |- computer03         |                                        |
| | |- nova-compute     | enabled :-) 2018-10-18T10:56:07.000000 |
| |- computer04         |                                        |
| | |- nova-compute     | enabled :-) 2018-10-18T10:56:05.000000 |
| |- computer08         |                                        |
| | |- nova-compute     | enabled :-) 2018-10-18T10:56:13.000000 |
| paas                  | available                              |
| |- computer05         |                                        |
| | |- nova-compute     | enabled :-) 2018-10-18T10:56:07.000000 |
| |- computer06         |                                        |
| | |- nova-compute     | enabled :-) 2018-10-18T10:56:13.000000 |
| |- computer07         |                                        |
| | |- nova-compute     | enabled :-) 2018-10-18T10:56:04.000000 |
+-----------------------+----------------------------------------+
```
由结果可以看出，该系统含有两个域： IMS 和 paas 。


但是该命令结果不太直观，不容易提取，我们也可以使用如下命令
```
nova aggregate-list

+----+---------+-------------------+
| Id | Name    | Availability Zone |
+----+---------+-------------------+
| 1  | IMS+RCS | IMS               |
| 4  | paas    | paas              |
+----+---------+-------------------+
```

根据 aggregate-list ，我们可以查看域对应的计算节点
```
nova aggregate-details paas

+----+------+-------------------+------------------------------------------+--------------------------+
| Id | Name | Availability Zone | Hosts                                    | Metadata                 |
+----+------+-------------------+------------------------------------------+--------------------------+
| 4  | paas | paas              | 'computer07', 'computer05', 'computer06' | 'availability_zone=paas' |
+----+------+-------------------+------------------------------------------+--------------------------+
```

由结果可知，paas域，一共包含三个computer节点，对于每个节点的使用情况，我们一样可以通过命令获取
```
nova hypervisor-show computer07

+---------------------------+------------+
| Property                  | Value      |
+---------------------------+------------+
| free_disk_gb              | 55         |
| free_ram_mb               | 126156     |
| host_ip                   | 193.2.0.37 |
| hypervisor_hostname       | computer07 |
| hypervisor_type           | QEMU       |
| id                        | 9          |
| local_gb                  | 445        |
| local_gb_used             | 390        |
| memory_mb                 | 257740     |
| memory_mb_used            | 131584     |
| npt_ept                   | ept        |
| pci_pools                 | -          |
| running_vms               | 8          |
| service_disabled_reason   | -          |
| service_host              | computer07 |
| service_id                | 27         |
| state                     | up         |
| status                    | enabled    |
| vcpus                     | 48         |
| vcpus_used                | 32         |
+---------------------------+------------+

#截取部分信息
```
我们可以得知，该节点的vcpu总数及其使用情况,memory_mb总数及其使用情况。

将每个节点的使用情况均获取后，经过计算，就可以得出域的资源整体使用情况。对于云平台的整体使用情况，我们也可以通过命令行获取：
```
nova hypervisor-stats

+----------------------+---------+
| Property             | Value   |
+----------------------+---------+
| count                | 6       |
| current_workload     | 0       |
| disk_available_least | 304     |
| free_disk_gb         | 518     |
| free_ram_mb          | 961736  |
| local_gb             | 2670    |
| local_gb_used        | 2152    |
| memory_mb            | 1546440 |
| memory_mb_used       | 584704  |
| running_vms          | 42      |
| vcpus                | 288     |
| vcpus_used           | 172     |
+----------------------+---------+
```


上面我们讨论了使用命令行进行相关信息的获取，下面我们讨论使用python-api进行相关信息的获取与计算。


## 2 使用 openstack python-api 获取域监控信息

程序代码如下：
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import json
from optparse import OptionParser
from novaclient import client as noclient
from novaclient import utils

#登录及授权
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    #获取云平台整体信息并打印
    total_info = nova_client.hypervisor_stats.statistics()._info.copy()
    print "total_info_vcpus:", total_info["vcpus"]
    print "total_info_vcpus_used:", total_info["vcpus_used"]
    print "total_info_memory_mb:", total_info["memory_mb"]
    print "total_info_memory_mb_used:", total_info["memory_mb_used"]
    print "total_info_running_vms:", total_info["running_vms"]

    #获取域列表信息
    aggregates = nova_client.aggregates.list()

    for aggregate in aggregates:
        #初始化每个域的资源统计变量
        vcpus = 0
        vcpus_used = 0
        memory_mb = 0
        memory_mb_used = 0
        running_vms = 0

        #获取每个aggregate信息，并保存对应的hostscomputer节点列表
        aggregate_info = aggregate._info.copy()
        print aggregate_info["id"], aggregate_info["name"], aggregate_info["availability_zone"], aggregate_info["hosts"]
        aggregate_hosts = aggregate_info["hosts"]

        #循环计算节点，保存相关资源信息
        for aggregate_host in aggregate_hosts:
            hypervisor_info = utils.find_resource(nova_client.hypervisors, aggregate_host)._info
            vcpus = vcpus + hypervisor_info["vcpus"]
            vcpus_used = vcpus_used + hypervisor_info["vcpus_used"]
            memory_mb = memory_mb + hypervisor_info["memory_mb"]
            memory_mb_used = memory_mb_used + hypervisor_info["memory_mb_used"]
            running_vms = running_vms + hypervisor_info["running_vms"]
        
        #打印域资源信息
        print "vcpus:", vcpus
        print "vcpus_used:", vcpus_used
        print "memory_mb:", memory_mb
        print "memory_mb_used:", memory_mb_used
        print "running_vms:", running_vms

if __name__ == "__main__":
    main()

```

执行该程序后，可以获取各个域节点的信息

```
total_info_vcpus: 288
total_info_vcpus_used: 172
total_info_memory_mb: 1546440
total_info_memory_mb_used: 584704
total_info_running_vms: 42

1 IMS+RCS IMS [u'computer04', u'computer03', u'computer08']
vcpus: 144
vcpus_used: 84
memory_mb: 773220
memory_mb_used: 243200
running_vms: 20

4 paas paas [u'computer07', u'computer05', u'computer06']
vcpus: 144
vcpus_used: 88
memory_mb: 773220
memory_mb_used: 341504
running_vms: 22
```

经过适当的计算，我们就可以获取各个域分配及使用比例等信息。


上面我们就使用 python-api 打印出了所有需要的信息，但是对于监控来说，我们需要提取的是各个监控项的信息，这样才能方便的搜索和做图表展示；下面我们讨论结合zabbix进行相关信息的监控。


## 3 结合zabbix获取域相关监控信息

### 3.1 获取可用域信息列表

上面我们已经获取了所有的可用域信息，但对于zabbix来说，我们还需要返回固定格式的数据，供zabbix进行解析：
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import json
from optparse import OptionParser
from novaclient import client as noclient
from novaclient import utils

#登录及授权
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    r = {"data":[]}
    aggregates = nova_client.aggregates.list()

    for aggregate in aggregates:
        aggregate_info = aggregate._info.copy()
        r['data'].append( {"{#NAME}":aggregate_info["name"]} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

if __name__ == "__main__":
    main()

```
执行后，返回的结果如下,可以通过供zabbix自动发现模版使用

```
{
  "data": [
    {
      "{#AVAILABLE_ZONE}": "IMS",
      "{#NAME}": "IMS+RCS"
    },
    {
      "{#AVAILABLE_ZONE}": "paas",
      "{#NAME}": "paas"
    }
  ]
}
```

### 3.3 zabbix，Item及自动发现LLD配置

云平台整体信息，Item设置如下：
```
openstack.total[vcpus]
openstack.total[vcpus_used]
openstack.total[memory_mb]
openstack.total[memory_mb_used]
openstack.total[running_vms]
```

获取{#NAME}后就可以根据监控项获取对应的监控内容了，zabbix自动发现自动发现key设置如下：

```
openstack.system.discovery
```

discovery下，设置监控Item,key设置如下：
```
openstack.zone[hosts,{#NAME}]
openstack.zone[vcpus,{#NAME}]
openstack.zone[vcpus_used,{#NAME}]
openstack.zone[memory_mb,{#NAME}]
openstack.zone[memory_mb_used,{#NAME}]
openstack.zone[running_vms,{#NAME}]
```
其中{#NAME}为第一步的脚本中返回的可用域对应的aggregate相关信息。

### 3.4 zabbix-agent配置与对应的脚本

对应的zabbix-agent.conf配置如下：

```
# /etc/zabbix/zabbix_agentd.d/userparameter_openstack-system.conf

UserParameter=openstack.system.discovery,python /etc/zabbix/zabbix_agentd.d/openstack-system.py --item discovery

UserParameter=openstack.total[*],python /etc/zabbix/zabbix_agentd.d/openstack-system.py --item total --moniter $1

UserParameter=openstack.zone[*],python /etc/zabbix/zabbix_agentd.d/openstack-system.py --item $1 --aggregate $2
```

对应的zabbix自动发现的监控脚本如下：
```
# /etc/zabbix/zabbix_agentd.d/openstack-system.py

#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import json
from optparse import OptionParser
from novaclient import client as noclient
from novaclient import utils

#登录及授权
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'])

def main():
    options = parse_args()
    if options.item=="discovery":
        zone_list()
    elif options.item=="total":
        total_moniter(options)
    else:
        zone_moniter(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "total", "hosts", "vcpus", "vcpus_used", "memory_mb", "memory_mb_used", "running_vms"]
    parser.add_option("", "--item", dest="item", help="", action="store", type="string", default=None)
    parser.add_option("", "--moniter", dest="moniter", help="", action="store", type="string", default=None)
    parser.add_option("", "--aggregate", dest="aggregate", help="", action="store", type="string", default=None)
    (options, args) = parser.parse_args()
    if options.item not in valid_item:
        parser.error("Item has to be one of: "+", ".join(valid_item))
    return options

#获取可用域列表
def zone_list():
    r = {"data":[]}
    aggregates = nova_client.aggregates.list()

    for aggregate in aggregates:
        aggregate_info = aggregate._info.copy()
        r['data'].append( {"{#NAME}":aggregate_info["name"], "{#AVAILABLE_ZONE}":aggregate_info["availability_zone"]} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取云平台整体监控信息
def total_moniter(options):
    total_info = nova_client.hypervisor_stats.statistics()._info.copy()
    print (total_info[options.moniter])

#获取可用域对应的监控信息
def zone_moniter(options):
    aggregate = utils.find_resource(nova_client.aggregates, options.aggregate)
    aggregate_info = aggregate._info.copy()
    aggregate_hosts = aggregate_info["hosts"]
    if options.item=="hosts":
        print (aggregate_hosts)
    else:
        monitor_data = 0
        for aggregate_host in aggregate_hosts:
            hypervisor_info = utils.find_resource(nova_client.hypervisors, aggregate_host)._info
            monitor_data = monitor_data + hypervisor_info[options.item]
        print (monitor_data)

if __name__ == "__main__":
    main()

```

在zabbix上进行对应的配置后重启，将模版应用于主机，此时应当监控获取所有的可用域，并监控对应的信息。

# 参考资料
1. Openstack 中的zone ,aggregates和host及其应用,https://blog.csdn.net/ztejiagn/article/details/8948688
2. nova 命令汇总四 ——计算相关命令,http://blog.51cto.com/13788458/2129157
3. The novaclient Python API,https://docs.openstack.org/python-novaclient/latest/reference/api/index.html
4. GitHub - larsks/openstack-api-samples,https://github.com/larsks/openstack-api-samples

# 如有疑问，欢迎交流
