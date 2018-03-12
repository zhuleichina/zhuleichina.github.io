---
layout: post
title: docker实践--使用docker部署wordpress
category: docker
tags: [docker]
description: 使用docker部署wordpress
---

# docker实践--使用docker部署wordpress

WordPress是一款个人博客系统，并逐步演化成一款内容管理系统软件，它是使用PHP语言和MySQL数据库开发的。用户可以在支持 PHP 和 MySQL数据库的服务器上使用自己的博客。

在搭建wordpress环境时，我们需要分别安装mysql/php/nginx/wordpress，本节我们学习使用docker简化wordpress的部署

## 1 镜像准备

需要下载镜像mysql/wordpress
```
docker pull mysql
docker pull wordpress
```

## 2 共享存储目录

mysql的数据文件夹为:
```
/var/lib/mysql
```
规划的本机共享存储目录为:
```
/home/docker/data/mysql
```
wordpress的代码文件夹为:
```
/var/www/html
```
规划的本机共享存储目录为:
```
/home/docker/data/wordpress
```

在宿主机上创建如上存储文件：
```
mkdir -p /home/docker/data/wordpress
mkdir -p /home/docker/data/mysql
chmod 777 -R /home/docker/
```

## 3 启动mysql与wordpress
```
#启动mysql
#-v配置了共享存储
#-e配置了root的密码
docker run --name wordpress_mysql --restart=always \
-e MYSQL_ROOT_PASSWORD="123456" \
-v /home/docker/data/mysql:/var/lib/mysql  \
-d \
mysql:latest

#启动worpress
#-v配置了共享存储
#-p配置为宿主机8080端口
docker run --name my_wordpress --restart=always \
--link wordpress_mysql:mysql \
-p 8080:80 \
-v /home/docker/data/wordpress:/var/www/html \
-d \
wordpress
```

## 4 wordpress的使用与配置
打开浏览器，输入如下地址进行访问：
```
192.168.1.1:8080

#192.168.1.1为宿主机IP地址
#8080为wordpress镜像共享的宿主机端口
```

## 5 wordpress镜像使用常见问题

### 5.1 wordpress默认附件大小

wordpress默认的附件大小为2M，一般的主题文件和照片等可能超过此大小而导致无法上传，需要增加如下配置文件

```
vim /home/docker/data/wp-config/uploads.ini

file_uploads = On
memory_limit = 64M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
```
在wordpress启动时，增加如下启动参数
```
-v /home/docker/data/wp-config/uploads.ini:/usr/local/etc/php/conf.d/uploads.ini \
```

### 5.2 wordpress绝对路径的问题

默认情况下，wordpress使用的是绝对路径，所以经过迁移或者IP映射后无法访问wordpress网站，可以修改wp-config.php配置文件

在共享存储的宿主机上：
```
vim /home/docker/data/wordpress/wp-config.php
```
在文件最后增加如下配置：
```
$home = 'http://'.$_SERVER['HTTP_HOST'];
$siteurl = 'http://'.$_SERVER['HTTP_HOST'];
define('WP_HOME', $home);
define('WP_SITEURL', $siteurl);
```

这样wordpress每次都会以当前浏览器访问的IP为WP_SITEURL，解决了wordpress迁移后无法访问的问题。

## 6 wordpress网站的迁移

使用docker部署wordpress，迁移后能够保证环境的一致性，不会出现各种环境问题。

### 6.1 镜像的导出
查看当前可用镜像
```
docker images

REPOSITORY                           TAG                 IMAGE ID            CREATED              SIZE
docker.io/wordpress                  latest              e8cebf03929c        6 days ago          407.2 MB
docker.io/mysql                      latest              f008d8ff927d        9 days ago          408.5 MB
```
导出镜像
```
docker save -o /opt/mysql.tar mysql:latest
docker save -o /opt/wordpress.tar wordpress:latest

#/opt/webapp.tar 导出镜像的位置及名称
```
我们查看/opt下文件列表，可以看到导出成功
```
ll /opt

-rw-------.  1 root root 417304064 Jan 25 19:16 wordpress.tar
-rw-------.  1 root root 415980032 Jan 25 19:11 mysql.tar
```

### 6.2 镜像的导入

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
docker load -i /opt/wordpress.tar
docker load -i /opt/mysql.tar
```
再次使用docker images，可以看到镜像已经导入成功，可以在当前未联网机器上使用。

### 6.3 共享存储

在新的宿主机上，在相同的位置建立共享存储文件夹，并将之前宿主机的文件拷贝过去，即可。

按上述步骤完成后，即可在新的宿主机上运行wordpress镜像，不需要修改其他参数即可保证运行环境的一致性。


# 参考资料

1. 用Docker搭建WordPress博客，http://blog.csdn.net/u011054333/article/details/70136099
2. Increase PHP file upload limit,https://github.com/docker-library/wordpress/issues/10
3. WordPress使用相对路径访问，http://blog.csdn.net/maxwoods/article/details/44895075
