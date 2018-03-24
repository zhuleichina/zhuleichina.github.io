---
layout: post
title: 使用logrotate进行日志文件备份
category: 日志
tags: [logrotate , 日志]
description: 部分软件日志会随着时间推移不断增大，对于分析和维护都造成很大的困难，因此需要日志管理进行处理。
---

# 使用logrotate进行日志文件备份

部分软件日志会随着时间推移不断增大，对于分析和维护都造成很大的困难，因此需要日志管理进行处理。

## 1. 需求分析

1. 每日将相关日志log文件进行备份

2. 存储最近10天的备份日志文件，超过10天的日志文件应能够自动删除

## 2. 实现过程分析

### 2.1 logrotate-linux日志总管

利用logrotate工具定时进行分割日志。

logrotate的配置文件为/etc/logrotate.conf，但是通常情况下，我们并不需要直接修改该配置文件。独立的日志文件定时任务都放在目录下：/etc/logrotate.d/



### 2.2 日志文件任务配置

新建文件位置及内容如下：

文件位置
```
/etc/logrotate.d/xxxxxx_log
```
文件内容
```
/var/log/xxxxxx.log {
daily
rotate 10
dateext
compress
delaycompress
missingok
notifempty
copytruncate
}
```
### 2.3 配置文件解析

daily:日志文件按天轮训存储，其值可以为：weekly、monthly、yearly

rotate 10:最多存储10天的日志，对于超过日期的日志予以删除

compress:对于历史归档予以压缩存储

delaycompress:与compress配合使用，最近一次的日志归档将不予压缩

missingok:忽略运行过程中的错误

notifempty:如果日志文件为空，轮训将不会进行

copytruncate：拷贝原始日志文件，并清空原日志文件


对于一般的Linux操作系统，以上配置完成后，cron每日定时任务即可自动执行logrotate日志备份。

# 如有疑问，欢迎交流
