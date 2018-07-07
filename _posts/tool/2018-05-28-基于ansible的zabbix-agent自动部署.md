---
layout: post
title: 基于ansible的zabbix-agent自动部署
category: 工具
tags: [ansible, zabbix]
description: 当遇到多台机器需要部署zabbix时，就需要使用自动化的方法，本文将介绍使用ansible部署agent。
---

# 基于ansible的zabbix-agent自动部署

当遇到多台机器需要部署zabbix时，就需要使用自动化的方法，本文将介绍使用ansible部署agent。

## 1 环境介绍

本次一共涉及到3台机器，一台server，两台agent机器：
```
#ansible管理端，zabbix-server机器
192.168.1.11

zabbix-agent机器：
192.168.1.12
192.168.1.13
```

## 2 ansible的安装与配置

ansible只需要在管理端进行安装即可，客户端需要支持ssh服务。

### 2.1 yum安装ansible
```
#yum安装ansible
yum install ansible -y
```

### 2.2 配置Linux主机ssh无密码访问

为了避免ansible下发命令时需要输入目标主机密码，可以通过证书签名达到ssh无密码访问。其中 ssh-keygen 生成一对密钥,通过 ssh-copy-id 来下发生成的公钥，具体操作如下：
```
#生成密钥对
ssh-keygen -t rsa

Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
58:68:a0:95:e9:bd:2d:86:ea:ce:2e:36:15:a8:a7:af root@localhost.localdomain
The key's randomart image is:
+--[ RSA 2048]----+
|    oo           |
|   oo. .         |
|  o. .o .        |
| . ....o         |
|.   ...oS        |
|. ... + .        |
| o.. . .         |
|o+.              |
|EBB              |
+-----------------+

#下发公钥，需要输入目标主机密码
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.1.12
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.1.13
```

### 2.3 配置主机与相关变量

ansible在/etc/ansible/中定义主机与变量
```
vim /etc/ansible/hosts

[zabbix-agent]
192.168.1.12 hostname=192.168.1.12
192.168.1.12 hostname=192.168.1.12

[zabbix-agent:vars]
server=192.168.1.11

#hostname定义为主机变量，只对一台主机生效
#zabbix-agent:vars 定义了zabbix-agent组变量，对zabbix-agent组生效
```

### 2.4 playbook任务列表

主控端当前目录文件为：
```
#agent配置文件模版
zabbix_agentd.conf.j2 

#agent,rpm安装包
zabbix-agent.rpm 

#agent,playbook剧本文件
zabbix-agent.yml
```

zabbix-agent.yml 文件内容为：
```
---
- hosts: zabbix-agent
  tasks:
  - name: 复制 zabbix-agent.rpm 安装文件
    copy:
      src: zabbix-agent.rpm
      dest: /opt/
      owner: root
      group: root
      mode: 0755

  - name: 目标主机安装 zabbix-agent.rpm
    yum:
      name: /opt/zabbix-agent.rpm
      state: present

  - name: 启动zabbix-agent服务
    service:
      name: zabbix-agent
      enabled: yes
      state: started

  - name: 配置zabbix_agentd.conf文件
    template:
      src: zabbix_agentd.conf.j2
      dest: /etc/zabbix/zabbix_agentd.conf

  - name: 重启zabbix-agent服务
    service:
      name: zabbix-agent
      state: restarted
  
```

zabbix_agentd.conf.j2 模版文件内容为：
```
# This is a configuration file for Zabbix agent daemon (Unix)
# To get more information about Zabbix, visit http://www.zabbix.com

PidFile=/var/run/zabbix/zabbix_agentd.pid

LogFile=/var/log/zabbix/zabbix_agentd.log

LogFileSize=0

Server= {{ server }}

ServerActive= {{ server }}

Hostname= {{ hostname }}
```

zabbix-agent动态配置下发用到了template 模版文件，{{ server }}在下发到目标主机时，会将变量server替换为hosts文件中定义的变量，从而实现的agent动态配置。

### 2.5 执行playbook

```
ansible-playbook zabbix-agent.yml

PLAY [zabbix-agent] *********************

TASK [Gathering Facts] ******************
ok: [192.168.1.12]
ok: [192.168.1.13]

TASK [复制 zabbix-agent.rpm 安装文件] *****
ok: [192.168.1.12]
ok: [192.168.1.13]

TASK [目标主机安装 zabbix-agent.rpm] *******************
ok: [192.168.1.12]
ok: [192.168.1.13]

TASK [启动zabbix-agent服务] ************
ok: [192.168.1.12]
ok: [192.168.1.13]

TASK [配置zabbix_agentd.conf文件] ************
ok: [192.168.1.12]
ok: [192.168.1.13]

TASK [重启zabbix-agent服务] ************
changed: [192.168.1.12]
changed: [192.168.1.13]

PLAY RECAP **************
192.168.1.12               : ok=6    changed=1    unreachable=0    failed=0
192.168.1.13               : ok=6    changed=1    unreachable=0    failed=0
```

可以看到，当增加很多台主机时，仅需要修改hosts文件，重新执行ansible-playbook即可完成agent的部署，结合zabbix自动发现功能，可以更加方便的实现自动监控功能

# 参考资料
1. Ansible常用模块介绍，https://blog.csdn.net/pushiqiang/article/details/78249665
2. Python自动化运维-技术与最佳实践 刘天斯 机械工业出版社
