---
layout: post
title: 基于ceilometer的openstack虚机资源监控2
category: 工具
tags: [openstack, zabbix, 虚机监控]
description: 在一些openstack环境中的虚机监控中还存在一定的问题。
---

# 基于ceilometer的openstack虚机资源监控2

在之前的文章 基于ceilometer的openstack虚机资源监控 中，我们讨论了如何使用ceilometer获取监控数据，使用zabbix监控脚本获取相关监控数据，但在一些openstack环境中的虚机监控中还存在一定的问题：
1. 在虚机数量过大(800+)时，监控项目过多，zabbix默认配置不满足使用要求
2. ceilometer查询脚本效率不高，查询会超时，获取不到监控数据
3. 各个虚机网卡数量不同，不易做展示分析

因为存在如上问题，所以需要对zabbix监控做对应的修改：
1. 修改zabbix_server与zabbix_agent默认配置，增加线程数，部分监控项修改为主动模式
2. ceilometer脚本查询后存文件中转，zabbix读取文件信息
3. 合并网卡流量计算，将虚机的所有网卡数据叠加，作为一个数据展示

## 1 zabbix配置信息修改

### 1.1 zabbix_server配置修改

```
/etc/zabbix/zabbix_server.conf

### Option: StartPollers
#	Number of pre-forked instances of pollers.
#   server主动模式的进程数量
StartPollers=60

### Option: StartTrappers
#	Number of pre-forked instances of trappers.
#	Trappers accept incoming connections from Zabbix sender, active agents and active proxies.
#	At least one trapper process must be running to display server availability and view queue
#	in the frontend.
#   接收zabbix sender,active agents的数据的进程数量
StartTrappers=50

### Option: StartDiscoverers
#	Number of pre-forked instances of discoverers.
#   处理自动发现的进程数量
StartDiscoverers=3

### Option: StartTimers
#	Number of pre-forked instances of timers.
#	Timers process time-based trigger functions and maintenance periods.
#	Only the first timer process handles the maintenance periods.
#   处理基于时间的trigger的进程数量
StartTimers=10

### Option: StartEscalators
#	Number of pre-forked instances of escalators.
#   处理escalators的进程数量
StartEscalators=10

### Option: HousekeepingFrequency
#	How often Zabbix will perform housekeeping procedure (in hours).
#	Housekeeping is removing outdated information from the database.
#	To prevent Housekeeper from being overloaded, no more than 4 times HousekeepingFrequency
#	hours of outdated information are deleted in one housekeeping cycle, for each item.
#	To lower load on server startup housekeeping is postponed for 30 minutes after server start.
#	With HousekeepingFrequency=0 the housekeeper can be only executed using the runtime control option.
#	In this case the period of outdated information deleted in one housekeeping cycle is 4 times the
#	period since the last housekeeping cycle, but not less than 4 hours and not greater than 4 days.
#   处理Housekeeping，删除无用信息的时间间隔
HousekeepingFrequency=24

### Option: MaxHousekeeperDelete
#	The table "housekeeper" contains "tasks" for housekeeping procedure in the format:
#	[housekeeperid], [tablename], [field], [value].
#	No more than 'MaxHousekeeperDelete' rows (corresponding to [tablename], [field], [value])
#	will be deleted per one task in one housekeeping cycle.
#	SQLite3 does not use this parameter, deletes all corresponding rows without a limit.
#	If set to 0 then no limit is used at all. In this case you must know what you are doing!
#   每次Housekeeping删除的最大行数
MaxHousekeeperDelete=1000000

### Option: CacheSize
#	Size of configuration cache, in bytes.
#	Shared memory size for storing host, item and trigger data.
#   处理Host Item Trigger 数据的内存大小
CacheSize=8G

### Option: StartDBSyncers
#	Number of pre-forked instances of DB Syncers.
#   处理数据库同步信息的进程数
StartDBSyncers=20

### Option: HistoryCacheSize
#	Size of history cache, in bytes.
#	Shared memory size for storing history data.
#   处理历史数据的内存大小
HistoryCacheSize=2G

### Option: HistoryIndexCacheSize
#	Size of history index cache, in bytes.
#	Shared memory size for indexing history cache.
#   处理历史数据索引的内存大小
HistoryIndexCacheSize=2G

### Option: TrendCacheSize
#	Size of trend cache, in bytes.
#	Shared memory size for storing trends data.
#   处理Trends数据的内存大小
TrendCacheSize=2G

### Option: ValueCacheSize
#	Size of history value cache, in bytes.
#	Shared memory size for caching item history data requests.
#	Setting to 0 disables value cache.
#   数据缓存在内存中的大小
ValueCacheSize=16G

### Option: UnreachablePeriod
#	After how many seconds of unreachability treat a host as unavailable.
#   Host默认为多久不可达
UnreachablePeriod=300

### Option: UnreachableDelay
#	How often host is checked for availability during the unreachability period, in seconds.
#   当Host不可达时，检测Host的频率
UnreachableDelay=60

### Option: AllowRoot
#	Allow the server to run as 'root'. If disabled and the server is started by 'root', the server
#	will try to switch to the user specified by the User configuration option instead.
#	Has no effect if started under a regular user.
#	0 - do not allow
#	1 - allow
#   是否允许以root用户执行
AllowRoot=1
```

### 1.2 zabbix_agent配置修改
```
/etc/zabbix/zabbix_agentd.conf


### Option: StartAgents
#	Number of pre-forked instances of zabbix_agentd that process passive checks.
#	If set to 0, disables passive checks and the agent will not listen on any TCP port.
#   agent被动模式的进程数量
StartAgents=100

### Option: RefreshActiveChecks
#	How often list of active checks is refreshed, in seconds.
#   检测主动检查Item的周期
RefreshActiveChecks=1800

### Option: BufferSend
#	Do not keep data longer than N seconds in buffer.
#   数据发送间隔
BufferSend=60

### Option: BufferSize
#	Maximum number of values in a memory buffer. The agent will send
#	all collected data to Zabbix Server or Proxy if the buffer is full.
#   内存中缓存的最大行数
BufferSize=200

### Option: AllowRoot
#	Allow the agent to run as 'root'. If disabled and the agent is started by 'root', the agent
#	will try to switch to the user specified by the User configuration option instead.
#	Has no effect if started under a regular user.
#	0 - do not allow
#	1 - allow
#   允许以root用户允许脚本等
AllowRoot=1
```
其他没有介绍的配置项，安装之前配置可以运行的即可。另外这样配置并没有做很详细的压力测试，仅表示在监控条数较多时，这样修改配置是可用的。

## 2 ceilometer查询信息存文件中转配置

### 2.1 将ceilometer查询信息存存为文件

/etc/zabbix/openstack/ceilometer.py

```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import logging
import threading
import json
import os
from ceilometerclient import client as cmclient
from novaclient import client as noclient

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

#限制运行运行的最大进程数，防止连接池过大，导致查询失败
maxThread=threading.Semaphore(100)

#日志处理模块初始化，记录处理过程
logging.basicConfig(
    level=logging.DEBUG,  # 定义输出到文件的log级别，大于此级别的都被输出
    format='%(asctime)s  %(filename)s : %(levelname)s  %(message)s',  # 定义输出log的格式
    datefmt='%Y-%m-%d %A %H:%M:%S',  # 时间
    filename='/etc/zabbix/openstack/multi_ceilometer.log',  # log文件名
    filemode='a')  # 写入模式“w”或“a”
    # Define a Handler and set a format which output to console
console = logging.StreamHandler()  # 定义console handler
console.setLevel(logging.INFO)  # 定义该handler级别
formatter = logging.Formatter('%(asctime)s  %(filename)s : %(levelname)s  %(message)s')  # 定义该handler格式
console.setFormatter(formatter)
# Create an instance
logging.getLogger().addHandler(console)  # 实例化添加handler

#需要添加新的监控项时，直接在以下list中添加即可，不需要修改主程序
#query_no为虚机的本身属性，不需要查询ceilometer
query_no = ["OS-EXT-AZ:availability_zone", "status"]
#query_num为ceilometer简单属性查询
query_num = ["cpu_util"]
#query_sum为ceilometer网卡属性查询，做了一个汇总的数据
query_sum = ["network.outgoing.packets", "network.incoming.packets"]

def myCeilometer(nova, n):
    with  maxThread:
        thradName = threading.currentThread().getName()
        logging.info(str(n) + " " + thradName + " " + nova.id + " " + "start")
        nets = nova.interface_list()
        nova_info = nova._info.copy()

        for query in query_no:
            dir_name = '/etc/zabbix/openstack/' + query
            if not os.path.exists(dir_name):
                os.mkdir(dir_name)
            state = "None"
            try:
                state = nova_info[query]
            except:
                state = "None2"

            file_name = '/etc/zabbix/openstack/' + query + '/' + nova.id + '.txt'
            logging.info(str(n) + " " + thradName + " " + nova.id + " " + query + " " + state)
            with open(file_name, 'w') as f:
                f.write(state)

        for query in query_num:
            dir_name = '/etc/zabbix/openstack/' + query
            if not os.path.exists(dir_name):
                os.mkdir(dir_name)
            num = 0
            try:
                fields = {'meter_name': query,
                        'q': [{"field": "resource_id", "op": "eq", "value": nova.id}],
                        'limit': 1}

                meters = ceilometer_client.samples.list(**fields)
                num = meters[0].counter_volume
            except IndexError,e:
                num = -1

            file_name = '/etc/zabbix/openstack/' + query + '/' + nova.id + '.txt'
            logging.info(str(n) + " " + thradName + " " + nova.id + " " + query + " " + str(num))
            with open(file_name, 'w') as f:
                f.write(str(num))

        for query in query_sum:
            dir_name = '/etc/zabbix/openstack/' + query
            if not os.path.exists(dir_name):
                os.mkdir(dir_name)
            num = 0
            for net in nets:
                net_info = net._info.copy()
                resource_id = nova_info["OS-EXT-SRV-ATTR:instance_name"] + "-" + nova.id + "-"  + "ovk"  + net_info["port_id"][0:11]
                try:
                    fields = {'meter_name': query,
                            'q': [{"field": "resource_id", "op": "eq", "value": resource_id}],
                            'limit': 1}
                    meters = ceilometer_client.samples.list(**fields)
                    num =  num + meters[0].counter_volume
                except IndexError,e:
                    num = -1
            file_name = '/etc/zabbix/openstack/' + query + '/'  + nova.id + '.txt'
            logging.info(str(n) + " " + thradName + " " + nova.id + " " + query + " " + str(num))
            with open(file_name, 'w') as f:
                f.write(str(num))

def main():
    n = 0
    novas = nova_client.servers.list(detailed='detailed', search_opts={'all_tenants': 1})
    #对每个虚机，启用一个线程，提高查询效率
    for nova in novas:
        n = n + 1
        a=threading.Thread(target=myCeilometer,args=(nova, n))
        a.start()

if __name__ == "__main__":
    main()
```
加入多线程前，处理完成800+虚机的各种信息大概需要3小时左右，加入多线程后，在100线程的并发下，处理完成所有信息大概需要15分钟。

### 2.2 将脚本加入定时周期任务

我们可以将该监本加入linux定时周期任务，每个小时执行一次。(每15分钟执行一次也行，根据自己需要进行调整)

```
crontab -e

0 */1 * * * python /etc/zabbix/openstack/multi_ceilometer.py
```

### 2.3 zabbix相关配置与查询脚本

zabbix自动发现配置
```
openstack.vm.discovery
```
zabbix自动发现Item
```
openstack.ceilometer[cpu_util,{#VMID}]
openstack.ceilometer[status,{#VMID}]
openstack.ceilometer["OS-EXT-AZ:availability_zone",{#VMID}]
openstack.ceilometer[network.incoming.packets,{#VMID}]
openstack.ceilometer[network.outgoing.packets,{#VMID}]
```
对于流量信息来说，我们获取的流量总包，但实际使用时，我们需要的是流量差值或者流量速率，因此在zabbix中将"Store value"设置为"Delta(speed per second)" 这样就可以不需要单独使用公式进行计算了。

zabbix-agent配置文件
```
UserParameter=openstack.vm.discovery,python /etc/zabbix/zabbix_agentd.d/openstack-vm.py --item discovery

UserParameter=openstack.ceilometer[*],python /etc/zabbix/zabbix_agentd.d/openstack-vm.py --item $1 --uuid $2
```

zabbix获取文件数据信息脚本
```
#!/usr/bin/python
# -*- coding: utf-8 -*-

import json
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

def main():
    options = parse_args()
    if options.item=="discovery":
        vm_list()
    else:
        ceilometer_query(options)

#判断入参合法性
def parse_args():
    parser = OptionParser()
    valid_item = ["discovery", "cpu_util", "vm_name_list", "status", "OS-EXT-AZ:availability_zone", "vcpus", "network.incoming.packets", "network.outgoing.packets"]
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
        nova_info = nova._info.copy()
        r['data'].append( {"{#VMNAME}":nova.name, "{#VMID}":nova.id, "{#VMZONE}":nova_info["OS-EXT-AZ:availability_zone"]} )
    print(json.dumps(r, indent=2, sort_keys=True, encoding="utf-8"))

#获取对应监控项的监控值
def ceilometer_query(options):
    cpu_util_file_name = '/etc/zabbix/openstack/' + options.item + '/' + options.uuid + '.txt'
    try:
        with open(cpu_util_file_name, 'r') as f:
            print f.read()
    except IOError,e:
        print -2

if __name__ == "__main__":
    main()

```

### 2.4 网卡监控包信息的计算

我们获取流量包的周期是小时，所以依靠zabbix设置，我们可以获取一小时内的流量速率packets/s，对于一天和一周内的流量速率，我们可以通过计算获得。

```
(avg("openstack.ceilometer[network.incoming.packets,{#VMID}]",#24))
(avg("openstack.ceilometer[network.incoming.packets,{#VMID}]",#168))
(avg("openstack.ceilometer[network.outgoing.packets,{#VMID}]",#24))
(avg("openstack.ceilometer[network.outgoing.packets,{#VMID}]",#168))
```
即24个小时点数据的平均值即为天平均数据，168个小时点数据的平均值即为周平均数据。

## 3 总结

对于监控项目来说，我们只关注虚机是否在使用的相关项目，所有只保留了CPU使用率和网卡流量的计算项目，对于其他项目并没有进行查询。但在启用了多线程且结果存文件的情况下，全部项目都进行监控也不会存在性能问题了，可以放心进行增加与裁剪。

对于Openstack虚机是否在使用上，监控虚机和流量也只能作为一种参考，还没有找到更好的监控方式，还需要继续进行研究。

# 参考资料
1. Ceilometer详解,https://blog.csdn.net/u010305706/article/details/51001622
2. The novaclient Python API,https://docs.openstack.org/python-novaclient/latest/reference/api/index.html
3. GitHub - larsks/openstack-api-samples,https://github.com/larsks/openstack-api-samples
4. Zabbix Server参数文件详解, https://www.linuxidc.com/Linux/2016-07/133242.htm

# 如有疑问，欢迎交流