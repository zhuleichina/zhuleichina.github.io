---
layout: post
title: docker实践--使用docker部署Jenkins
category: docker
tags: [docker,jenkins]
description: 使用docker部署Jenkins
---

# docker实践--使用docker部署Jenkins

Jenkins是一个开源软件项目，是基于Java开发的一种持续集成工具，用于监控持续重复的工作，旨在提供一个开放易用的软件平台，使软件的持续集成变成可能。本文介绍如何使用docker部署Jenkins

## 1 镜像准备

```
docker pull jenkins/jenkins:lts
```

## 2 启动与运行jenkins容器

### 2.1 启动容器
```
docker run --name jenkins --restart=always \
-p 8080:8080 \
-p 50000:50000 \
-d \
jenkins/jenkins:lts
```

### 2.2 共享存储目录

jenkins容器中/var/jenkins_home目录存放着Jenkins的插件和配置，在运行时，最好配置共享存储目录。
```
docker run --name jenkins --restart=always \
-p 8080:8080 \
-p 50000:50000 \
-v /home/docker/jenkins_home:/var/jenkins_home \
-d \
jenkins/jenkins:lts
```

此时打开浏览器如下地址，即可访问Jenkins
```
192.168.1.1:8080
# 192.168.1.1为宿主机IP地址
```

## 3 Jenkins镜像的导出与导入

### 3.1 镜像的导出
查看当前可用镜像
```
docker images

REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
docker.io/jenkins/jenkins   lts                 6382bf759f2a        12 days ago         814.6 MB
```
导出该镜像
```
docker save -o /opt/jenkins.tar docker.io/jenkins/jenkins:lts
# /opt/jenkins.tar 导出镜像的位置及名称
# docker.io/jenkins/jenkins:lts 导出镜像的REPOSITORY与TAG
```
我们查看/opt下文件列表，可以看到导出成功
```
ll /opt
-rw-------   1 root root 839046656 Mar 28 14:36 jenkins.tar
```

### 3.2 镜像的导入
在已经安装docker，但没有相关配置的机器上执行：
```
#添加开机启动
systemctl enable docker
#开启docker进程
systemctl start docker
```
查看当前可用镜像
```
docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```
可以看到镜像为空，导入镜像：
```
docker load -i /opt/jenkins.tar
# /opt/jenkins.tar为刚刚导出的镜像
```
再次使用docker images，可以看到镜像已经导入成功。
```
docker images

REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
docker.io/jenkins/jenkins   lts                 6382bf759f2a        12 days ago         814.6 MB
```

## 4 Jenkins重新部署与迁移

对于使用了docker的Jenkins来说，因为我们已经做了存储的映射，，拷贝宿主机上如下目录，然后重新部署Jenkins容器即可。
```
/home/docker/jenkins_home
```
因映射是存储在宿主机上的，为了防止数据丢失，可以使用云备份。

# 参考资料
1. jenkinsci/docker,https://github.com/jenkinsci/docker/blob/master/README.md

# 如有疑问，欢迎交流
