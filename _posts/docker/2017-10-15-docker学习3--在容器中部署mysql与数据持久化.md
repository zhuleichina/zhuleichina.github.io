---
layout: post
title: docker学习3--在容器中部署mysql与数据持久化
category: docker
tags: [docker]
description: 通过上一节的学习，我们知道了如何部署一个不带数据库的静态nginx页面；但一般的web应用中，还需要部署mysql数据库，本节我们将学习如何使用容器部署mysql数据库。
---

# docker学习3--在容器中部署mysql与数据持久化

通过上一节的学习，我们知道了如何部署一个不带数据库的静态nginx页面；但一般的web应用中，还需要部署mysql数据库，本节我们将学习如何使用容器部署mysql数据库。

## 1 mysql独立部署

我们可以将mysql与web应用部署在同一个容器内，但更一般的用法是将mysql独立部署一个容器。

```
#获取mysql5.6.36官方镜像（mysql5.7变动较大，推荐使用5.6）
docker pull mysql:5.6.36
```

我们可以进入mysql:5.6.36容器进行mysql远程登录的相关设置。
```
#运行mysql:5.6.36容器,-p映射为宿主机3306端口
docker run -it -p 3306:3306 mysql:5.6.36 /bin/bash
#开启mysql进程
root@0950cf64b8e6:/# service mysql start

#进入mysql
root@0950cf64b8e6:/# mysql

#修改root用户密码为123456
mysql> update user set password=password("123456") where user='root';

#允许远程用户访问（一般应当设置为白名单IP，此处为所有IP）
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;

#验证是否设置成功，host中含有%
mysql> select host, user from user;

#保存退出
mysql> flush privileges;
mysql> exit;

#停止容器后，保存当前镜像为webmysql
root@0950cf64b8e6:/# docker stop 0950cf64b8e6
root@0950cf64b8e6:/# docker commit 0950 webmysql
```

以上设置完成后，我们在宿主机上安装mysql客户端，就可以通过宿主机的IP进行mysql数据库的使用了。

## 2 mysql数据持久化

对于容器数据库来说，一旦容器停止，容器中的数据就会消失，不利于数据存储，虽然我们可以通过定时commit的方法来保存容器中的数据，但我们有更好的实现方法。

### 使用-v共享存储 

mysql默认的数据存储目录为/var/lib/mysql,我们可以通过宿主机共享容器/var/lib/mysql目录的方式来实现数据的持久化。

```
#带-v参数启动webmysql
docker run -it -v /var/mysql/data:/var/lib/mysql -p 3306:3306 mysql:5.6.36 /bin/bash
```
镜像启动后，我们启动mysql服务，发现mysql无法启动。查找官方文档，因SElinux服务开启，需要在宿主机执行如下命令：
```
chcon -Rt svirt_sandbox_file_t /var/mysql/data
```
或者关闭SElinux也可以。
```
#临时关闭
setenforce 0
#修改配置文件，需要重启
vim /etc/selinux/config 
SELINUX=disabled
```
上述配置完成后，仍然无法启动mysql，结合mysql日志查看可能是文件权限的问题，在宿主机上给予共享文件夹对应的权限：
```
#赋予本地存储对应的权限，单读写权限不行
chmod 777 -R /var/mysql/data/
```
设置完成后，容器可以启动mysql服务。

在宿主机查看/var/mysql/data/文件夹下，发现已经将/var/lib/mysql/文件夹内容同步，使用stop关闭容器后，文件夹数据不会消失。再次启动容器mysql后，数据库内容仍然存在。数据持久化设置完成。

## 3 连接mysql容器

### 3.1 mycentos容器使用link连接

启动mysql容器
```
docker run --name=mysql_server -it -v /var/mysql/data:/var/lib/mysql -p 3306:3306 webmysql /bin/bash
#--name=mysql_server指定了容器运行的name
```
启动mycentos容器
```
docker run --link=mysql_server:db -it -p 80:80 mycentos /bin/bash
#--link=mysql_server:db，指定了能够与mysql数据库容器继续连接，db指定了一个连接的别名
```
在mycentos上安装mysql客户端后就可以使用命令行登录mysql：
```
mysql -h db -uroot -p123456
MySQL [(none)]>
```
在web应用的配置文件中，更改数据库的配置即可：
```
host: db
username: root
password: 123456 
```

### 3.2 宿主机使用IP连接

部分情况下，我们可能需要使用宿主机连接登录mysql容器，这样显然不能使用link的方法。

容器与宿主机之间是通过bridge进行的网络连接，我们可以通过使用内网IP地址连接容器mysql。
```
#查看webmysql容器的IP地址，e79dd0dc4f1f为其docker ps显示的ID
docker inspect --format '{{ .NetworkSettings.IPAddress }}' e79dd0dc4f1f
172.17.0.2
```
其地址为172.17.0.2，我们可以使用该IP地址登录容器mysql
```
mysql -h 172.17.0.2 -uroot -p123456
MySQL [(none)]>
```

值得注意的是，这种使用IP的方法也适用于容器与容器之间的mysql的连接，容器连接宿主机mysql。

## 4 更多主题探讨

通过这几节的学习，我们能够使用容器部署网站与数据库，然而对于docker技术而言，这只是其中最基础的使用。以下是与web部署强相关的主题：

1. 通过commit，我们能够保存对容器的更改存储在宿主机，但当宿主机出现问题时，就需要进行使用镜像恢复。这涉及到如何备份与恢复images镜像。
2. 我们创建容器mycentos与webmysql，都是通过手动的方式，而docker更一般的用法是使用Dockerfile，我们可以尝试这种更简便的使用方法。

# 参考资料
1. MySQL 官方 Docker 镜像的使用，https://www.cnblogs.com/cfrost/p/6241892.html
2. 自己学Docker:8.容器的持久化,http://blog.csdn.net/mungo/article/details/51472130
3. mysql,https://hub.docker.com/_/mysql/
4. 查看 SELinux状态及关闭SELinux,http://blog.51cto.com/bguncle/957315
5. docker容器链接宿主机mysql，https://segmentfault.com/a/1190000008701796
6. Docker中容器的备份、恢复和迁移，http://www.linuxidc.com/Linux/2015-08/121184.htm
7. Docker使用link建立容器之间的连接，http://www.jianshu.com/p/13752117ff97

# 如有疑问，欢迎交流
