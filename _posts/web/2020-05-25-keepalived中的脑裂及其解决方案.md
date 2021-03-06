---
layout: post
title: keepalived中的脑裂及其解决方案
category: web服务
tags: [keepalived, 脑裂]
description: keepalived中的脑裂及其解决方案
---


# keepalived中的脑裂及其解决方案

## 1. 脑裂问题

在keepalived双机高可用系统中，当心跳断链后，本来是一个整体的HA系统，就分裂成两个独立的个体。由于互相失去了联系，都认为是对方出现了故障，两个独立的节点就发生了脑裂，互相争夺资源，包括VIP、共享存储等，其会导致一系列问题，两台主机同时具备VIP，实际只会有一台起作用。两台主机同时读写共享存储，可能会导致数据损坏等。

## 2. 解决方案

以下解决方案适用于物理机，对于虚机的心跳断链，未有相关资料进行参考。

### 2.1 增加冗余心跳线

增加冗余的心跳线，减少出现问题的几率。

备注：如希望尽可能简单，建议两根心跳线均直连。

### 2.2 设置仲裁机制

如设置网关为参考IP，当心跳断链时，两个节点各自ping一下参考IP，不通则表示断链发生在本端，应当放弃主动竞争，不去抢占相关的资源如VIP、共享磁盘等。

更进一步，可以让ping不通参考IP的一方尝试重启，如果不通则继续重启，如果心跳恢复则成为备机。

备注：心跳线需要经过交换机才能使用这种仲裁IP。建议设置两个心跳，一根直连，一根经交换机，额外再加上这种仲裁机制。

### 2.3 启用磁盘锁

正在服务一方锁住共享磁盘，心跳断链时，另一方抢不走被加锁的磁盘资源。

但使用锁磁盘也会有一个不小的问题，如果占用共享盘的一方不主动“解锁”，另一方就永远得不到共享磁盘。如服务器死机或者崩溃等，就会发生这种情况。

### 2.4 增加监控告警机制

当备机抢占相关资源后，及时发生监控告警，进行人工仲裁处理。

# 参考资料
1. keepalived中的脑裂, https://www.cnblogs.com/clsn/p/7748553.html
2. keepalived产生脑裂的原因与解决方法, https://blog.csdn.net/WanJiaBaoBao/article/details/83715748
