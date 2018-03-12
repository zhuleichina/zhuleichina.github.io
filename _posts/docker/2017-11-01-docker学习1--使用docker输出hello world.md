---
layout: post
title: docker学习1--使用docker输出hello world
category: docker
tags: [docker]
description: Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的Linux机器上，也可以实现虚拟化，容器是完全使用沙箱机制，相互之间不会有任何接口。
---

# docker学习1--使用docker输出hello world

## 1.什么是docker
Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的Linux机器上，也可以实现虚拟化，容器是完全使用沙箱机制，相互之间不会有任何接口。

### docker使用场景
docker有很多用途，目前对于我来说，可以预期的场景为：

1. 提高开发效率：一般的开发工作中，开发环境的搭建是件头疼的事情，每个开发人员都得重复搭一套一致的环境，使用docker后，可以搭建一次后存为镜像，其他团队成员就可以直接使用了。
2. 快速部署：在虚拟机之前，引入新的硬件资源需要消耗几天的时间。Docker的虚拟化技术将这个时间降到了几分钟，Docker只是创建一个容器进程而无需启动操作系统，这个过程只需要秒级的时间。

更多用途介绍，见参考资料5：Docker 的应用场景在哪里？

## 2. docker安装与启动

一般的Linux发行版本中，已经预装了docker，输入如下命令确认是否已经预装docker
```
docker --version
#如果已经存在会输出当前docker版本
Docker version 1.10.3, build 694b432-unsupported
```
可以使用yum升级到最新版本
```
yum update docker
```
如果当前系统中不存在docker，可以使用yum安装
```
yum install docker
```
docker的启动停止命令如下：
```
#启动
service docker start
#停止
service docker stop
#重启
service docker restart
```
## 3. 搜索与下载镜像
### 3.1 查找可用镜像
```
#例如查找centos的镜像
docker search centos

#结果如下组织：
INDEX  NAME  DESCRIPTION  STARS OFFICIAL  AUTOMATED
#STARS：镜像的星数，一般选择星数高的下载
#OFFICIAL：是否是官方的镜像，如果有一般选择官方镜像
```
### 3.2 下载可用镜像
```
#下载centos的官方镜像
docker pull centos
```
因docker默认使用官方镜像源，速度很慢，所以我们一般可以选择更换镜像源。

### 3.3 配置国内镜像加速
登录阿里云镜像服务，https://cr.console.aliyun.com/?spm=5176.100239.blogcont29941.12.eJkJlD#/accelerator，获取自己的加速器地址(专用镜像加速地址，需要自己获取)

按照页面帮助文档，修改配置文件/etc/docker/daemon.json
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://abcdefg.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
配置完成后，重新启动docker发现，docker无法启动。
```
service docker restart

Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
```
依据参考资料1，需要centos/redhat上配置其他文件。
```
#删除错误的配置文件
rm /etc/docker/daemon.json

#重新配置
vim /etc/sysconfig/docker
#OPTIONS中增加registry-mirror属性
OPTIONS='--selinux-enabled --log-driver=journald --registry-mirror=https://abcdefg.mirror.aliyuncs.com'
DOCKER_CERT_PATH=/etc/docker
```
配置完成后，重新启动docker，查看启动参数。
```
#重启
service docker restart
#查看启动信息
ps aux | grep docker
/usr/bin/dockerd-current --log-driver=journald --registry-mirror=https://abcdefg.mirror.aliyuncs.com
```
再次使用docker pull centos 下载centos镜像，可以明显看到速度有很大的提升。

### 3.4 查看当前已存在的镜像
```
docker images
#结果如下：
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/centos    latest              3fa822599e10        8 days ago          203.5 MB

```
## 4. 运行centos镜像输出hello world
对于程序员来说，入门一种技术最关键的一步来了，使用docker run输出hello world。
```
docker run centos echo "hello word"

hello word
```
可以看到，docker run有两个参数，一个是镜像名，一个是要在镜像中运行的命令。

当echo命令运行结束后，容器也会随之停止，如果需要一直打开容器的控制台，可以输入如下命令：
```
docker run -it centos /bin/bash
[root@1a3d1376e367 /]# 

#可以看到，终端上已经由centos容器控制台接管，此时直接echo输出
[root@1a3d1376e367 /]# echo "hello word"
hello word
[root@1a3d1376e367 /]# 

#exit退出容器
[root@1a3d1376e367 /]# exit
exit
[root@localhost opt]# 
```


# 参考资料
1. 在阿里云上使用 Docker 并配置阿里云镜像加速器，结果遇到 daemon.json 导致 docker daemon 无法启动的问题，https://pagespeed.v2ex.com/t/326229
2. docker: Error response from daemon: Container command could not be invoked..，http://blog.csdn.net/qq_29352959/article/details/54847794
3. Docker 镜像加速器,https://yq.aliyun.com/articles/29941
4. Docker，https://baike.baidu.com/item/Docker/13344470?fr=aladdin
5. Docker 的应用场景在哪里？,https://www.zhihu.com/question/22969309
6. Docker入门教程,http://www.docker.org.cn/book/docker/what-is-docker-16.html

# 如有疑问，欢迎交流
