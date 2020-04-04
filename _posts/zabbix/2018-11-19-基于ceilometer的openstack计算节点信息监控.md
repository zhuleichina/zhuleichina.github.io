---
layout: post
title: 基于ceilometer的openstack计算节点信息监控
category: 工具
tags: [openstack, zabbix, 计算节点监控]
description: 使用ceilometer对计算节点的使用信息进行监控。
---

# 基于ceilometer的openstack计算节点信息监控

在之前的文章中，我们讨论了可用域的资源使用情况监控、虚机的使用情况监控，计算节点作为可用域的组成部分与基本分配单位，本文将讨论使用ceilometer对计算节点的使用信息进行监控。

## 1 计算节点信息监控文件中转

本次,我们主要监控了计算节点的如下信息
```
#作为hypervisor本身的属性，不需要使用ceilometer进行查询
query_no = ["status", "state", "vcpus", "vcpus_used", "running_vms"]
#网卡信息，相同的硬件，网卡信息是一致的，我这里有两种计算节点的硬件，所以net_adds有两种需要尝试，如果不知道可以使用ceilometer meter-list进行查询
query_net = ["compute.node.network.incoming.packets", "compute.node.network.outgoing.packets"]
```
作为计算节点来说，还有内存使用信息，磁盘使用信息监控等，可以根据项目需求进行增加与裁剪，修改对应的List即可，不需要修改主程序。

监控脚本及文件存储
/etc/zabbix/hyper/multi_hyper.py
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports

import logging
import threading
import json
import os
from optparse import OptionParser

from ceilometerclient import client as cmclient
from novaclient import client as noclient
from novaclient import utils

#getting the credentials
keystone = {}
keystone['os_username']='admin'
keystone['os_password']='keystone'
keystone['os_auth_url']='http://lb-vip:5000/v2.0/'
keystone['os_tenant_name']='admin'
keystone['os_cacert']='/home/tecs/ssl/certs/ca.pem'

#creating an authenticated client
ceilometer_client = cmclient.get_client(2,**keystone)
nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'], cacert=keystone['os_cacert'])

maxThread=threading.Semaphore(10)

base_dir = '/etc/zabbix/hyper/'
#作为hypervisor本身的属性，不需要使用ceilometer进行查询
query_no = ["status", "state", "vcpus", "vcpus_used", "running_vms"]
#网卡信息，相同的硬件，网卡信息是一致的，我这里有两种计算节点的硬件，所以net_adds有两种需要尝试，如果不知道可以使用ceilometer meter-list进行查询
query_net = ["compute.node.network.incoming.packets", "compute.node.network.outgoing.packets"]
net_adds = ["ens1f0", "ens1f1", "ens2f0", "ens2f1", "ens3f0", "ens3f1", "ens3f2", "ens3f3"]
net_add_rs = ["eno1", "eno2", "eno3", "eno4", "ens7f0", "ens7f1", "ens8f0", "ens8f1"]

logging.basicConfig(
    level=logging.DEBUG,  # 定义输出到文件的log级别，大于此级别的都被输出
    format='%(asctime)s  %(filename)s : %(levelname)s  %(message)s',  # 定义输出log的格式
    datefmt='%Y-%m-%d %A %H:%M:%S',  # 时间
    filename=base_dir +'hyper.log',  # log文件名
    filemode='a')  # 写入模式“w”或“a”
# Define a Handler and set a format which output to console
console = logging.StreamHandler()  # 定义console handler
console.setLevel(logging.INFO)  # 定义该handler级别
formatter = logging.Formatter('%(asctime)s  %(filename)s : %(levelname)s  %(message)s')  # 定义该handler格式
console.setFormatter(formatter)
# Create an instance
logging.getLogger().addHandler(console)  # 实例化添加handler

def myCeilometer(hypervisor, n):
    with  maxThread:
        thradName = threading.currentThread().getName()
        logging.info(str(n) + " " + thradName + " " + hypervisor.hypervisor_hostname + " " + "start")

        hypervisor_info = utils.find_resource(nova_client.hypervisors, hypervisor.hypervisor_hostname)._info.copy()
        for query in query_no:
            dir_name = base_dir + query
            if not os.path.exists(dir_name):
                os.mkdir(dir_name)
            file_name = dir_name + '/'  + hypervisor.hypervisor_hostname + '.txt'
            data = hypervisor_info[query]
            logging.info(str(n) + " "  + hypervisor.hypervisor_hostname + " " + query + " " + str(data))
            with open(file_name, 'w') as f:
                f.write(str(data))

        for query in query_net:
            dir_name = base_dir + query
            if not os.path.exists(dir_name):
                os.mkdir(dir_name)
            num = 0
            try:
                num = -1
                for net_add in net_adds:
                    resource_id = hypervisor.hypervisor_hostname + "-" + net_add
                    
                    fields = {'meter_name': query,
                            'q': [{"field": "resource_id", "op": "eq", "value": resource_id}],
                            'limit': 1}
                    meters = ceilometer_client.samples.list(**fields)
                    num =  num + meters[0].counter_volume
            except IndexError,e:
                num = -2
                for net_add in net_add_rs:
                    resource_id = hypervisor.hypervisor_hostname + "-" + net_add
                    
                    fields = {'meter_name': query,
                            'q': [{"field": "resource_id", "op": "eq", "value": resource_id}],
                            'limit': 1}
                    meters = ceilometer_client.samples.list(**fields)
                    num =  num + meters[0].counter_volume
            file_name = dir_name + '/'  + hypervisor.hypervisor_hostname + '.txt'
            logging.info(str(n) + " "  + hypervisor.hypervisor_hostname + " " + query + " " + str(num))
            with open(file_name, 'w') as f:
                f.write(str(num))

        logging.info(str(n) + " " + thradName + " " + hypervisor.hypervisor_hostname + " " + "end")

#所属域的信息需要用aggregate进行反查，比较特殊
def zone_query():
    n = 0
    dir_name = base_dir + 'zone'
    if not os.path.exists(dir_name):
        os.mkdir(dir_name)
    aggregates = nova_client.aggregates.list()
    for aggregate in aggregates:
        n = n + 1
        for host in aggregate.hosts:
            file_name = dir_name + '/'  + host + '.txt'
            logging.info(str(n) + " "  + host + " " + aggregate.name)
            with open(file_name, 'w') as f:
                f.write(aggregate.name)
    logging.info(str(n) + " "  + "zone_query" + " " + "end")

def main():
    zone_query()
    n = 0
    hypervisors = nova_client.hypervisors.list(False)
    for hypervisor in hypervisors:
        n = n + 1
        a=threading.Thread(target=myCeilometer,args=(hypervisor, n))
        a.start()

if __name__ == "__main__":
    main()

```

## 2 将脚本加入定时周期任务
我们可以将该监本加入linux定时周期任务，每个小时执行一次。(每15分钟执行一次也行，根据自己需要进行调整，极限可以设置为5分钟也可以)

```
crontab -e

30 */1 * * * python /etc/zabbix/hyper/multi_hyper.py
```

## 3 zabbix相关配置与查询脚本

### 3.1 zabbix自动发现配置与Item
zabbix自动发现配置
```
openstack.hyper.discovery
```
zabbix自动发现Item
```
openstack.hyper[compute.node.network.incoming.packets,{#HOSTNAME}]
openstack.hyper[compute.node.network.outgoing.packets,{#HOSTNAME}]
openstack.hyper["running_vms",{#HOSTNAME}]
openstack.hyper["state",{#HOSTNAME}]
openstack.hyper["status",{#HOSTNAME}]
openstack.hyper["vcpus",{#HOSTNAME}]
openstack.hyper["vcpus_used",{#HOSTNAME}]
openstack.hyper["zone",{#HOSTNAME}]
```
对于流量信息来说，我们获取的流量总包，但实际使用时，我们需要的是流量差值或者流量速率，因此在zabbix中将"Store value"设置为"Delta(speed per second)" 这样就可以不需要单独使用公式进行计算了。

### 3.2 zabbix-agent配置文件
```
UserParameter=openstack.hyper.discovery,python /etc/zabbix/zabbix_agentd.d/openstack-hyper.py --item discovery

UserParameter=openstack.hyper[*],python /etc/zabbix/zabbix_agentd.d/openstack-hyper.py --item $1 --hostname $2
```

### 3.3 zabbix数据查询脚本
/etc/zabbix/zabbix_agentd.d/openstack-hyper.py
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
keystone['os_cacert']='/home/tecs/ssl/certs/ca.pem'

nova_client = noclient.Client(2, keystone['os_username'], keystone['os_password'], keystone['os_tenant_name'], keystone['os_auth_url'], insecure=False, cacert=keystone['os_cacert'])

def main():
    options = parse_args()
    if options.item=="discovery":
        hyper_list()
    else:
        ceilometer_query(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "hyper_name_list", "zone", "compute.node.network.incoming.packets", "compute.node.network.outgoing.packets", "vcpus", "vcpus_used", "running_vms", "status", "state"]
    parser.add_option("", "--item", dest="item", help="", action="store", type="string", default=None)
    parser.add_option("", "--hostname", dest="hostname", help="", action="store", type="string", default=None)
    (options, args) = parser.parse_args()
    if options.item not in valid_item:
        parser.error("Item has to be one of: "+", ".join(valid_item))
    return options

#获取可用域列表
def hyper_list():
    r = {"data":[]}
    hypervisors = nova_client.hypervisors.list(False)
    for hypervisor in hypervisors:
        r['data'].append( {"{#HOSTNAME}":hypervisor.hypervisor_hostname} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取对应监控项的监控值
def ceilometer_query(options):
    cpu_util_file_name = '/etc/zabbix/hyper/' + options.item + '/' + options.hostname + '.txt'
    try:
        with open(cpu_util_file_name, 'r') as f:
            print f.read()
    except IOError,e:
        print -2

if __name__ == "__main__":
    main()

```
### 3.4 网卡监控包信息的计算

我们获取流量包的周期是小时，所以依靠zabbix设置，我们可以获取一小时内的流量速率packets/s，对于一天和一周内的流量速率，我们可以通过计算获得。

```
(avg("openstack.hyper[compute.node.network.incoming.packets,{#HOSTNAME}]",#24))
(avg("openstack.hyper[compute.node.network.incoming.packets,{#HOSTNAME}]",#168))
(avg("openstack.hyper[compute.node.network.outgoing.packets,{#HOSTNAME}]",#24))
(avg("openstack.hyper[compute.node.network.outgoing.packets,{#HOSTNAME}]",#168))
```
即24个小时点数据的平均值即为天平均数据，168个小时点数据的平均值即为周平均数据。

## 4 总结

我们使用了多线程与存文件的方式进行了计算节点相关信息的监控，对于辅助监控当前计算节点是否在使用可以作为一定的参考。

# 参考资料
1. Ceilometer详解,https://blog.csdn.net/u010305706/article/details/51001622
2. The novaclient Python API,https://docs.openstack.org/python-novaclient/latest/reference/api/index.html
3. GitHub - larsks/openstack-api-samples,https://github.com/larsks/openstack-api-samples

# 如有疑问，欢迎交流
