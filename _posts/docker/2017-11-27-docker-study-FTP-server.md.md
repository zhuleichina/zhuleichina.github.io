---
layout: post
title: docker实践--使用docker搭建FTP服务器
category: docker
tags: [docker]
description:使用docker搭建FTP服务器
---

# docker实践--使用docker搭建FTP服务器

## 1 镜像准备

```
docker pull stilliard/pure-ftpd:hardened
```

## 2 启动与运行FTP容器
### 2.1 启动容器
```
docker run -d --name ftpd_server --restart=always \
-p 8021:21 \
-p 30000-30209:30000-30209 \
-e "ADDED_FLAGS=-O w3c:/var/log/pure-ftpd/transfer.log" \
-e "PUBLICHOST=localhost" \
-v /home/docker/ftp/user:/etc/pure-ftpd/passwd \
-v /home/docker/ftp/file:/home/ftpusers \
-v /home/docker/ftp/log:/var/log/pure-ftpd \
stilliard/pure-ftpd:hardened
```

### 2.2 需要操作FTP容器
```
docker exec -it ftpd_server /bin/bash
```
### 2.3 退出FTP容器命令行
```
CTRL+P+Q
```
### 2.4 停止FTP服务器的运行
```
docker stop ftpd_server
```

## 3 FTP容器的常用操作

### 3.1 FTP容器添加用户与共享文件夹
```
pure-pw useradd bob -f /etc/pure-ftpd/passwd/pureftpd.passwd -m -u ftpuser -d /home/ftpusers/bob
#终端会提醒需要输入两次密码
#新加用户名为bob，密码为自己输入的密码
#允许访问的文件夹为/home/ftpusers/bob，-v已经映射为宿主机/home/docker/ftp/file/bob
```

### 3.2 验证FTP连接

宿主机上输入
```
ftp -p localhost 8021
```
输入用户名密码即可进入FTP命令行

使用filezilla等FTP工具，IP地址为宿主机的内外网IP地址，端口为8021，再输入用户名密码即可连接。

### 3.3 共享文件
在宿主机/home/docker/ftp/file/bob目录下将需要共享的文件放进去

### 3.4 修改密码
```
pure-pw passwd bob -f /etc/pure-ftpd/passwd/pureftpd.passwd -m
#再次输入两次密码
```
### 3.5 删除用户
```
pure-pw userdel bob -f /etc/pure-ftpd/passwd/pureftpd.passwd -m
#bob为需要删除的用户
```
删除用户时，共享的文件仍然存在在宿主机/home/docker/ftp/file/bob文件夹下，可以根据需要决定是否保留

### 3.6 日志文件

日志文件位置为：/home/docker/ftp/log/transfer.log


## 4 镜像的导出与导入

对于不能联网的机器，是不能使用docker pull从网上直接拉镜像的，这个时候就需要使用容器的导出与导入功能了。

### 4.1 镜像的导出
查看当前可用镜像
```
docker images

REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
docker.io/stilliard/pure-ftpd             hardened            ddec1e99475e        7 weeks ago         437.5 MB
```
导出该镜像
```
docker save -o /opt/ftp.tar stilliard/pure-ftpd:hardened
#/opt/ftp.tar 导出镜像的位置及名称
#stilliard/pure-ftpd:hardened 导出镜像的REPOSITORY与TAG
```
我们查看/opt下文件列表，可以看到导出成功
```
ll /opt
-rw-------   1 root root 453882880 Dec 16 14:02 ftp.tar
```

### 4.2 镜像的导入
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
docker load -i /opt/ftp.tar
#/opt/ftp.tar为刚刚导出的镜像
```
再次使用docker images，可以看到镜像已经导入成功，可以在当前未联网机器上使用。

# 参考资料
1. stilliard/pure-ftpd,https://hub.docker.com/r/stilliard/pure-ftpd/
2. Docker使用pure-ftp的方法及配置,https://www.cnblogs.com/HD/p/5664394.html
3. Docker images导出和导入，http://www.jianshu.com/p/8408e06b7273
4. 在 docker 之间导出导入镜像，http://blog.csdn.net/a906998248/article/details/46236687

# 如有疑问，欢迎交流