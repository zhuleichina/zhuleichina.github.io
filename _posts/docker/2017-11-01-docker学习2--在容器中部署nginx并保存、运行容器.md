---
layout: post
title: docker学习2--在容器中部署nginx并保存、运行容器
category: docker
tags: [docker]
description: 在上一节，我们学到如何使用centos容器输出hello world，本节我们将学习如何在镜像中安装nginx并保存更改，运行自己的容器，并学习如何进行端口映射与后台运行容器。
---

# docker学习2--在容器中部署nginx并保存、运行容器

在上一节，我们学到如何使用centos容器输出hello world，本节我们将学习如何在镜像中安装nginx并保存更改，运行自己的容器，并学习如何进行端口映射与后台运行容器。

 ## 1 共享本地存储

```
#-v共享本地存储
docker run -it -v /opt:/opt centos /bin/bash
```
通过-v参数，冒号前为宿主机目录，必须为绝对路径，冒号后为镜像内挂载的路径。

此时，可以查看容器的/opt目录是否已经共享本地存储
```
[root@2b4accd4cb25 /]# ll /opt

```
当需要使用本机文件的时候可以复制到本机/opt目录，这样容器就可以共享了。

## 2 使用yum安装nginx并启动
```
yum -y install nginx
```
不修改配置文件，直接启动nginx，并访问80端口
```
#启动nginx
nginx
#访问nginx网站80端口
curl localhost:80
```
这样我们即在容器内启动了nginx默认页面在80端口上，通过curl可以查看其页面代码。

## 3 保存对容器的更改

首先，查看当前运行中的容器
```
#在宿主机中执行
docker ps
#记录结果CONTAINER ID
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
2b4accd4cb25        centos              "/bin/bash"         About an hour ago   Up About an hour                        mad_wozniak
```

保存该容器
```
docker commit 2b4a mycentos

#2b4a为CONTAINER ID的前四位
#mycentos是自己更改镜像后的别名
```

查看本地已有容器
```
docker images
#结果如下：
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mycentos            latest              19604df31369        9 seconds ago       375.4 MB
docker.io/centos    latest              3fa822599e10        8 days ago          203.5 MB
```
可以看到,mycentos已经作为一个新的容器保存。

## 4 运行mycentos容器
```
docker run -it -v /opt:/opt mycentos /bin/bash

#进入mycentos后直接运行nginx，验证nginx是否已经安装
[root@16f46f4e363d /]# nginx
[root@16f46f4e363d /]# ps aux |grep nginx
root         14  0.0  0.1 122912  2108 ?        Ss   01:35   0:00 nginx: master process nginx
nginx        15  0.0  0.1 123296  3128 ?        S    01:35   0:00 nginx: worker process
root         17  0.0  0.0   9

#可以验证，容器已经正确保存修改并重新运行。
```

## 5 端口映射与后台运行

### 5.1 端口映射

我们在容器mycentos中启动了nginx服务并开启了80端口，对于更一般的情况来说，我们需要在容器外，也就是宿主机开启对应的映射端口，这样才能对外提供网站服务。我们可以通过如下命令开启映射：
```
docker run -it -p 80:80 mycentos /bin/bash
[root@8e2455671b01 /]# nginx
#开启Nginx进程后，在宿主机浏览器使用localhost:80即可访问nginx默认页面
# -p 参数 host_port:port ，host_port指定了宿主机的端口，后面是容器的端口
#-p 还可以开启多端口映射 -p host_port1:port1 -p host_port2:port2
```

### 5.2 开启nginx并后台运行容器

对于一般的网站部署来说，我们并不需要开启bash窗口，我们需要容器开启nginx后一直在后台运行就可以了，我们可以输入如下命令开启容器后台运行：
```
#前台开启
docker run -it -p 80:80 mycentos /bin/bash
[root@8e2455671b01 /]# nginx
#按住 Ctrl + P + Q 退出容器，进入后台运行
#此时使用 docker ps 
docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                NAMES
8c2299567a7d        mycentos            "/bin/bash"         30 seconds ago      Up 27 seconds       0.0.0.0:80->80/tcp   boring_goldberg
```
可以看到容器已经保持在后台运行,我们可以对这个在后台的容器进行一定的操作
```
#进入刚刚进入后台运行的容器
docker attach 8c2299567a7d
#验证Nginx进程
[root@8c2299567a7d /]# ps aux | grep nginx
#按住 Ctrl + P + Q 退出容器，进入后台运行

#后台开启容器mycentos，但不进入容器
docker run -dit -p 80:80 mycentos /bin/bash
#后台执行nginx，不进入容器
docker exec -it 8c2299567a7d nginx

#退出后台运行
docker stop 8c2299567a7d
```



# 参考资料
1. 详解Docker挂载本地目录及实现文件共享,http://blog.csdn.net/magerguo/article/details/72514813
2. 在linux命令下如何访问一个url？,http://blog.csdn.net/zhuying_linux/article/details/6881728
3. 保存对容器的修改,http://www.docker.org.cn/book/docker/docer-save-changes-10.html
4. Docker学习笔记-Docker端口映射,http://blog.csdn.net/qq_29994609/article/details/51730640
5. Docker学习笔记（四）之容器查看启动终止删除,http://blog.csdn.net/u013246898/article/details/52945884
6. 保持后台运行 Docker 容器，https://www.douban.com/note/602752252/
7. docker 后台运行和进入后台运行的容器，https://www.cnblogs.com/hanxing/p/7832178.html

# 如有疑问，欢迎交流