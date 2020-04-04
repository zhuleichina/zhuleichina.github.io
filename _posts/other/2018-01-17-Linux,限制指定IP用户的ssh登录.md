---
layout: post
title: Linux,限制指定IP用户的ssh登录
category: 未分类
tags: [ssh ,安全]
description: RedHat、CGSL等Linux系统中，默认是开启ssh允许所有IP进行登录的，对于密码泄漏和互联网攻击等存在很大的危险。
---

# Linux,限制指定IP用户的ssh登录

RedHat、CGSL等Linux系统中，默认是开启ssh允许所有IP进行登录的，对于密码泄漏和互联网攻击等存在很大的危险。

对于一般的需要ssh的远程的机器，我们可以开启指定IP可ssh的访问限制，增加相关的安全性。

以下示例表示允许IP地址为192.168.1.2/3的IP地址进行ssh登录:

## 1. hosts.allow文件添加允许远程访问的IP地址

```
vi etc/hosts.allow

#
# hosts.allow This file contains access rules which are used to
# allow or deny connections to network services that
# either use the tcp_wrappers library or that have been
# started through a tcp_wrappers-enabled xinetd.
#
# See 'man 5 hosts_options' and 'man 5 hosts_access'
# for information on rule syntax.
# See 'man tcpd' for information on tcp_wrappers
#
sshd:192.168.1.2:allow
sshd:192.168.1.3:allow
```

## 2. hosts.deny中设置其余所有IP禁止访问

```
vi etc/hosts.deny

#
# hosts.deny This file contains access rules which are used to
# deny connections to network services that either use
# the tcp_wrappers library or that have been
# started through a tcp_wrappers-enabled xinetd.
#
# The rules in this file can also be set up in
# /etc/hosts.allow with a 'deny' option instead.
#
# See 'man 5 hosts_options' and 'man 5 hosts_access'
# for information on rule syntax.
# See 'man tcpd' for information on tcp_wrappers
#
sshd:ALL
```

## 3. 验证配置效果

a.使用IP列表内的IP仍然可以正常登录，不受影响；

b.使用IP列表外的IP登录此机器会报错：
```
[root@localhost ~]# ssh root@192.168.1.1
ssh_exchange_identification: read: Connection reset by peer
[root@localhost ~]#
```
配置成功。

# 如有疑问，欢迎交流
