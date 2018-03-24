---
layout: post
title: kibana可视化报警插件KAAE的安装与配置
category: 日志
tags: [ELK , KAAE]
description: 前面我们学习了如何进行ELK的安装与使用，那么我们如何才能实现ELK的邮件告警，本节我们将介绍如何使用kibana的可视化插件KAAE进行邮件告警。
---

# kibana可视化报警插件KAAE的安装与配置
前面我们学习了如何进行ELK的安装与使用，那么我们如何才能实现ELK的邮件告警，本节我们将介绍如何使用kibana的可视化插件KAAE进行邮件告警。

## 1 KAAE的介绍
Kibi & Kibana Alerting & Reporting App for Elasticsearch

github地址：https://github.com/sirensolutions/sentinl

sentinl在Kibana日志可视化的基础上，扩展了kibana的日志监控和告警的功能。

## 2 KAAE的安装

sentinl.zip的版本需要与kibana的版本一致。

### 2.1 方式一：在线安装
针对kibana5.6.4
```
/usr/share/kibana/bin/kibana-plugin install https://github.com/sirensolutions/sentinl/releases/download/tag-5.6.2/sentinl-v5.6.4.zip
```

### 2.2 方式二：编译安装

第一种方式其他版本不一定能够使用，且速度极其缓慢，一般采用编译安装。

```
# 1.安装npm
yum -y install npm
# 输入 npm -v 会显示对应的版本号则代表安装成功

# 2.升级openssl
yum -y update openssl

# 3.安装gulp
npm install gulp -g
# 输入 gulp -v 显示对应的版本号则代表安装成功

# 4.clone源码
git clone https://github.com/sirensolutions/sentinl

# 5.编译安装包
cd sentinl && npm install && gulp package

# 6.安装sentinl
/usr/share/kibana/bin/kibana-plugin install file://`pwd`/target/gulp/sentinl.zip

# 也可以使用别人已经编译好的对应版本的sentinl
/usr/share/kibana/bin/kibana-plugin install file:///opt/sentinl/target/gulp/sentinl.zip
```
安装成功后重启elasticsearch和kibana，打开kibana界面，会发现界面菜单多了Sentinl菜单。
```
# 浏览器打开如下地址,其中192.168.0.1为服务器的IP
192.168.0.1:5601
```

### 2.3 引用在线css的问题

对于部分未联网的机器，使用Sentinl时，会时不时的跳出错误页面，因源文件代码中引用了在线bootstrap.min.css，需要进行注释
```
# 文件位置如下：
/usr/share/kibana/optimize/bundles/sentinl.style.css
/usr/share/kibana/plugins/sentinl/public/style/main.less
# 源文件第一行如下：
@import url(//maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css);
# 需要使用 /* 这是需要注释的内容 */进行注释
/*@import url(//maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css);*/
```

## 3 SENTINL的配置

### 3.1 kibana配置邮件服务器

如需发送邮件应该首先申请对应的邮件服务器，然后在kibana配置文件最后增加如下内容：
```
vim /etc/kibana/kibana.yml

sentinl:
  settings:
    email:
      active: true
      user: email_user
      password: email_password
      host: email_host
      ssl: false
      timeout: 10000
    report:
      active: true
      tmp_path: /tmp/

# email_user/email_password/email_host均由邮件服务器管理员提供
# ssl: false 公司邮件服务器需要设置为false
```

### 3.2 配置监视器Watchers

浏览器打开sentinl界面
```
192.168.0.1:5601/app/sentinl
```
打开菜单 "New" -->> "+ Watcher" ，配置项如下：
```
General
# Title:监视器的名字
# Schedule:多少时间执行一次

Input
# 需要执行什么查询语句

Condition
# 告警条件是什么

Actions
# 如何通知用户告警信息，我们一般配置为邮件
```
以下我们简单介绍各个配置的使用：

#### Schedule
```
# 每分钟执行一次
every 1 mins
# 没2消失执行一次
every 2 hours
```

#### Input

Input定义了查询条件
```
# 如下查询语句定义了查询在两分钟内/var/log/nginx/error.log的新增日志
"query": {
          "bool": {
            "must": [
              {
                "match": {
                  "source": "/var/log/nginx/error.log"
                }
              },
              {
                "range": {
                  "@timestamp": {
                    "gte": "now-2m",
                    "lte": "now",
                    "format": "epoch_millis"
                  }
                }
              }
            ]
          }
        }

# 与elasticsearch查询语法一致
```

#### Condition

Condition定义了告警的条件
```
# 如下语句定义了上述查询总个数大于100条就告警
{
  "script": {
    "script": "payload.hits.total > 100"
  }
}
```

#### Actions
默认告警方式为邮件，我们可以对邮件进行如下配置：
```
Throttle
# 如果发生了告警邮件，暂停多长时间不再发送相同的告警邮件
# 界面可以配置，分别为时、分、秒

Priority
# 告警等级，界面可配置为 High/Medium/Low

Title
# 邮件动作的名称

To
# 发送给谁邮件

From
# 发件方是谁

Subject
# 邮件主题

Body
# 邮件正文
```

### 3.3 配置实例

配置完成后，在Raw里可以查看生产的对应配置文件

```
# 如下配置实例分开进行解析
# schedule
# "later": "every 1 mins"，每分钟执行一次

# input
# 查询两分钟内日志/var/log/nginx/error.log新增的记录

# condition
# 记录数大于100条就发出告警

# action
# 发送邮件告警，1小时内不重复发送

{
  "_index": "watcher",
  "_type": "sentinl-watcher",
  "_id": "v5tgrlakfrk-ooi72ox1aeh-3tawuhcbtkv",
  "_version": 22,
  "found": true,
  "_source": {
    "title": "watcher_nginx_error_log",
    "disable": true,
    "report": false,
    "trigger": {
      "schedule": {
        "later": "every 1 mins"
      }
    },
    "input": {
      "search": {
        "request": {
          "index": [
            "filebeat-*"
          ],
          "body": {
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "source": "/var/log/nginx/error.log"
                    }
                  },
                  {
                    "range": {
                      "@timestamp": {
                        "gte": "now-2m",
                        "lte": "now",
                        "format": "epoch_millis"
                      }
                    }
                  }
                ]
              }
            }
          }
        }
      }
    },
    "condition": {
      "script": {
        "script": "payload.hits.total >= 10"
      }
    },
    "actions": {
      "email_me": {
        "throttle_period": "1h0m0s",
        "email_html": {
          "to": "zhangsan@zte.com.cn,lisi@zte.com.cn,
          "from": "log_analysis@zte.com.cn",
          "subject": "[log_analysis] nginx_error_log",
          "body": "openlab_error_log:\nFound {{payload.hits.total}} Events
          "save_payload": true
        }
      }
    },
    "transform": {}
  }
}
```
如果出现问题，需要查询错误日志
```
/var/log/kibana/kibana.stderr
```
正常日志记录：
```
/var/log/kibana/kibana.stdout
```

# 参考资料

1. 基于Kibana的可视化监控报警插件 KAAE 的配置,http://blog.csdn.net/whg18526080015/article/details/73812400
1. Kibana 可视化监控报警插件 KAAE 的介绍与使用,http://blog.csdn.net/phachon/article/details/53424631#comments
1. sentinl/wiki,https://github.com/sirensolutions/sentinl/wiki

# 如有疑问，欢迎交流