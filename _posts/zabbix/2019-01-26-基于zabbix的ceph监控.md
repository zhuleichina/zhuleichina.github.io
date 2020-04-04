---
layout: post
title: 基于zabbix的ceph监控
category: zabbix
tags: [openstack, zabbix, ceph]
description: 我们对于ceph的监控需求包括ceph的状态和ceph,osd中in和up的比例，因为比较简单，我们可以自己实现。
---

# 基于zabbix的ceph监控

我们对于ceph的监控需求包括ceph的状态和ceph,osd中in和up的比例，因为比较简单，我们可以自己实现。

## 1 控制台获取ceph相关结果

我们可以通过控制台获取ceph的状态
```
ceph health

#对应的正常结果为
HEALTH_OK

#异常结果为：
HEALTH_WARN
HEALTH_ERR
```
通过控制台我们还可以获取OSD的相关状态
```
ceph osd stat

#对应结果为：
50 osds: 50 up, 50 in
flags noscrub,nodeep-scrub
```
控制台还可以获取ceph的总使用比例的相关信息
```
GLOBAL:
    SIZE     AVAIL     RAW USED     %RAW USED
    182T      154T       29100G         15.54
POOLS:
    NAME             ID     USED      %USED     MAX AVAIL     OBJECTS
    5GC-CINDER       1      5192G     10.46        44446G     1363562
    TECS6-CINDER     3      9233G     28.85        22768G     2396157
```

## 2 zabbix配置文件

```
UserParameter=ceph.health,ceph health
UserParameter=ceph.health.number,python /etc/zabbix/zabbix_agentd.d/ceph.py

UserParameter=ceph.osd.total, ceph osd stat | awk '/osd/{print $1}'
UserParameter=ceph.osd.up, ceph osd stat | awk '/osd/{print $3}'
UserParameter=ceph.osd.in, ceph osd stat | awk '/osd/{print $5}'

UserParameter=ceph.global.total, ceph df |awk '/182T/{print $1}'
UserParameter=ceph.global.available, ceph df |awk '/182T/{print $2}'
UserParameter=ceph.global.used, ceph df |awk '/182T/{print $3}'
UserParameter=ceph.global.used.ratio, ceph df |awk '/182T/{print $4}'
```

其中ceph.health.number为ceph状态对应的数字，是为了给grafana做展示使用的。

## 3 对应脚本文件

```
#!/usr/bin/python
# -*- coding: utf-8 -*-

#imports
import commands

def main():
    ceph_status = commands.getoutput("ceph health")
    if ceph_status.startswith("HEALTH_OK"):
        print 0
    elif ceph_status.startswith("HEALTH_WARN"):
        print 1
    elif ceph_status.startswith("HEALTH_ERR"):
        print 2
    else:
        print -1

if __name__ == "__main__":
    main()
```
即当ceph正常时，返回0，异常时返回1，错误时返回2。

## 4 zabbix其他设置

我们监控了OSD的IN和UP的个数，但对于告警来说，我们还需要设置如下监控项
```
100*last("ceph.osd.in")/last("ceph.osd.total")
100*last("ceph.osd.up")/last("ceph.osd.total")
```
即in和up的比例，这样对应的Trigger就可以设置为比例不为100%：
```
{Template Ceph Health:UpRatio.count(#2,100,"ne")}=2
{Template Ceph Health:InRatio.count(#2,100,"ne")}=2
```
同时还应当设置如下Trigger
```
#已用ceph大于80%
{Template Ceph Health:ceph.global.used.ratio.count(#2,80,"ge")}=2

#ceph状态WARN告警
{Template Ceph Health:ceph.health.count(#2,"HEALTH_WARN","like")}=2

#ceph状态ERROR告警
{Template Ceph Health:ceph.health.count(#2,"HEALTH_ERR","like")}=2
```

## 5 总结

使用如上监控方法我们可以简单的监控ceph的状态和磁盘的总体利用率等，对于更深入的监控，我们可以参考网上的模版，做更深入的ceph监控。具体见参考资料。

# 参考资料

1. thelan/ceph-zabbix, https://github.com/thelan/ceph-zabbix
