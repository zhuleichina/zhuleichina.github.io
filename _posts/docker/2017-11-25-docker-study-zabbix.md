---
layout: post
title: docker实践--使用docker部署zabbix
category: docker
tags: [docker]
description:使用docker部署zabbix
---

# docker实践--使用docker部署zabbix

## 1 zabbix服务器端配置

在zabbix-server服务器上进行如下配置：

### 1.1 镜像准备

下载镜像：mysql、zabbix/zabbix-server-mysql、zabbix/zabbix-web-nginx-mysql
```
docker pull mysql:5.6.36
docker pull zabbix/zabbix-server-mysql
docker pull zabbix/zabbix-web-nginx-mysql
```

### 1.2 共享存储目录
```
mkdir -p /home/docker/data/mysql
mkdir -p /home/docker/data/zabbix/usr
mkdir -p /home/docker/data/zabbix/var
chmod 777 -R /home/docker/
```

### 1.3 启动mysql:5.6.36
```
docker run --name zabbix_mysql --hostname zabbix_mysql --restart=always \
-e MYSQL_ROOT_PASSWORD="123456" \
-e MYSQL_USER="zabbix" \
-e MYSQL_PASSWORD="123456" \
-e MYSQL_DATABASE="zabbix" \
-p 3306:3306  \
-v /home/docker/data/mysql:/var/lib/mysql  \
-d \
mysql:5.6.36
```

### 1.4 启动zabbix_server
```
docker run  --name zabbix_server  --restart=always \
--link zabbix_mysql:mysql \
-e DB_SERVER_HOST="mysql" \
-e MYSQL_USER="zabbix" \
-e MYSQL_DATABASE="zabbix" \
-e MYSQL_PASSWORD="123456" \
-v /etc/localtime:/etc/localtime:ro \
-v /home/docker/data/zabbix/usr:/usr/lib/zabbix \
-v /home/docker/data/zabbix/var:/var/lib/zabbix \
-p 10051:10051 \
-d \
zabbix/zabbix-server-mysql
```

### 1.5 启动zabbix_nginx_web
```
docker run --name zabbix_web --restart=always \
--link zabbix_mysql:mysql \
--link zabbix_server:zabbix_server \
-e DB_SERVER_HOST="mysql" \
-e MYSQL_USER="zabbix" \
-e MYSQL_PASSWORD="123456" \
-e MYSQL_DATABASE="zabbix" \
-e ZBX_SERVER_HOST="zabbix_server" \
-e PHP_TZ="Asia/Shanghai" \
-p 80:80 \
-p 8443:443 \
-d \
zabbix/zabbix-web-nginx-mysql
```

此时浏览器访问宿主机80端口，即可使用zabbix-server服务
```
#浏览器打开如下地址，其中192.168.1.1为zabbix服务器IP
192.168.1.1/zabbix
```

## 2 zabbix-agent配置

在需要监控的机器上安装agent进行如下配置：

### 2.1 镜像准备

下载镜像：zabbix/zabbix-agent
```
docker pull zabbix/zabbix-agent
```

### 2.2 共享存储目录
```
mkdir -p /home/docker/data/zabbix/conf
mkdir -p /home/docker/data/zabbix/var
chmod 777 -R /home/docker/
```

### 2.3 启动zabbix-agent
```
#172.17.0.1为本机zabbix，当不是本机时，更改为zabbix-server的IP地址
docker run --name zabbix_agent --restart=always \
-p 10050:10050 \
-e ZBX_HOSTNAME="zabbix_agent" \
-e ZBX_SERVER_HOST="172.17.0.1" \
-e ZBX_SERVER_PORT=10051 \
-v /home/docker/data/zabbix/var:/var/lib/zabbix \
-v /home/docker/data/zabbix/conf:/etc/zabbix/zabbix_agentd.d \
-d \
zabbix/zabbix-agent

#ZBX_HOSTNAME配置为zabbix-server中添加的对应的agent的主机名称
```

此时，可以在zabbix-server上加入刚刚启动的agent，接下来就是配置zabbix了，安装完成

## 3 docker开机启动

### 3.1 docker服务加入开机启动
```
systemctl enable docker
#成功加入开机启动会有如下提示：
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

### 3.2 docker容器开机启动
docker run 启动参数中加入--restart=always 参数，在容器出现重启等情况退出时，会自动重启，不需要再单独设置容器启动脚本。

## 4 镜像的导出与导入

对于不能联网的机器，是不能使用docker pull从网上直接拉镜像的，这个时候就需要使用容器的导出与导入功能了。

### 4.1 镜像的导出
查看当前可用镜像
```
docker images

REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
docker.io/zabbix/zabbix-agent             latest              19e75a9513da        23 hours ago        56.67 MB
docker.io/zabbix/zabbix-server-mysql      latest              3abad97a6d75        24 hours ago        107.3 MB
docker.io/zabbix/zabbix-web-nginx-mysql   latest              039292d2eae1        26 hours ago        176.8 MB
docker.io/mysql                           5.6.36              d9ad3d6d1a44        5 months ago        298.3 MB
```
导出镜像
```
docker save -o /opt/zabbix_agent.tar zabbix/zabbix-agent:latest
#/opt/zabbix_agent.tar 导出镜像的位置及名称
#zabbix/zabbix-agent:latest 导出镜像的REPOSITORY与TAG
docker save -o /opt/zabbix_server.tar zabbix/zabbix-server-mysql:latest
docker save -o /opt/zabbix_nginx.tar zabbix/zabbix-web-nginx-mysql:latest
docker save -o /opt/zabbix_mysql.tar mysql:5.6.36
```
我们查看/opt下文件列表，可以看到导出成功
```
ll /opt
-rw-------   1 root root  61114368 Dec 16 14:54 zabbix_agent.tar
-rw-------   1 root root 305571840 Dec 16 15:02 zabbix_mysql.tar
-rw-------   1 root root 182195200 Dec 16 15:01 zabbix_nginx.tar
-rw-------   1 root root 112269824 Dec 16 15:00 zabbix_server.tar
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
docker load -i /opt/zabbix_agent.tar
#/opt/zabbix_agent.tar为刚刚导出的镜像
docker load -i /opt/zabbix_server.tar
docker load -i /opt/zabbix_nginx.tar
docker load -i /opt/zabbix_mysql.tar
```
再次使用docker images，可以看到镜像已经导入成功，可以在当前未联网机器上使用。

## 5 zabbix相关日志

在容器中没有专门的日志文件，所以没有做日志文件存储映射，我们可以通过如下命令查看zabbix的运行日志：
```
docker logs zabbix_server
docker logs zabbix_agent
docker logs zabbix_web
docker logs zabbix_mysql
```

## 6 zabbix重新部署与迁移

对于使用了docker的zabbix来说，因为我们已经做了存储的映射，所以数据库和zabbix的配置可以很容器的迁移到其他机器上，拷贝宿主机上如下目录，然后重新部署zabbix容器即可。
```
/home/docker/data/zabbix
/home/docker/data/mysql
```
因映射是存储在宿主机上的，为了防止数据丢失，可以使用云备份。

# 参考资料
1. docker 安装 zabbix,http://blog.csdn.net/u012373815/article/details/71598457
2. docker搭建zabbix,https://www.cnblogs.com/Dicky-Zhang/p/7189714.html
3. Docker下实战zabbix三部曲之二：监控其他机器,http://blog.csdn.net/boling_cavalry/article/details/77095153
4. [经验分享] Docker容器开机自动启动（在宿主机重启后或者Docker服务重启后），https://www.iyunv.com/thread-181637-1-1.html
5. Docker images导出和导入，http://www.jianshu.com/p/8408e06b7273
6. 在 docker 之间导出导入镜像，http://blog.csdn.net/a906998248/article/details/46236687

# 如有疑问，欢迎交流
