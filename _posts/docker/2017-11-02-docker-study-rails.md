---
layout: post
title: docker实践--使用docker部署rails网站
category: docker
tags: [docker]
description: 使用docker部署rails网站
---

# docker实践--使用docker部署rails网站

基于Rails开发的网站，在部署上比较耗时，在之前的文章中,详细介绍了其部署的具体过程。

Docker的典型应用场景之一就包括web网站的部署，本文将介绍基于Dockerfile的Rails网站的部署。

## 1 准备镜像

首先我们需要明确自己的Ruby版本，比如我使用的是：ruby2.1
```
docker pull phusion/passenger-ruby21:0.9.27
#如果使用ruby2.2，可以使用：
#docker pull phusion/passenger-ruby22:0.9.27
#0.9.27指的是版本号，在生产环境中部署建议指定版本号
#其他问题参见参考资料1的官方文档
```

## 2 Dockerfile文件

### 2.1 Dockerfile文件的创建

Dockerfile文件可以在基础镜像的基础上自动构建新的镜像，在生产环境中一般很少使用commit保存新镜像，一般都是使用Dockerfile文件。

创建Dcokerfile文件时，一般将其放入一个新建的空目录下:
```
mkdir myweb
cd myweb
touch Dockerfile
```
一般情况下我们也可以选择将配置文件和网站代码编译进镜像文件，最终的myweb目录结构如下：
```
|--myweb
|    |--Dockerfile
|    |--webapp.conf  #nginx网站配置文件
|    |--webapp   #网站代码文件夹
|         |--public   #网站图片、视频等静态文件
|         |    |--………………
|         |--config   #网站相关配置文件
|         |    |--database.yml  #数据库配置
|         |    |--config.yml    #全局配置
|         |    |--secrets.yml   #密钥文件
|         |    |--………………
|         |--………………

# database.yml需要指定数据库IP地址 host
# secrets.yml 需要指定网站生产环境密钥
# config.yml  指定需要使用的全局变量
```

### 2.2 Dockerfile文件编写
```
vim Dockerfile

# FROM指定了基础镜像
# select the version of ruby
FROM phusion/passenger-ruby21:0.9.27

# ENV设置环境变量
# Set correct environment variables.
ENV HOME /root

# CMD执行镜像初始化
# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# 清理临时文件
# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# nginx默认是关闭的，开启nginx
# Nginx and Passenger are disabled by default. Enable them like so:
RUN rm -f /etc/service/nginx/down

# 将网站的代码和配置文件放在镜像里
# Adding your web app to the image
RUN rm /etc/nginx/sites-enabled/default
ADD webapp.conf /etc/nginx/sites-enabled/webapp.conf
RUN mkdir /home/app/webapp
ADD /webapp/ /home/app/webapp/

# 更新镜像内的系统
# To upgrade the OS in the image
#RUN apt-get update && apt-get upgrade -y -o Dpkg::Options::="--force-confold"

# bundle install与数据库初始化
# Bundle install
WORKDIR /home/app/webapp
RUN bundle install
RUN bundle exec rake assets:precompile db:migrate RAILS_ENV=production
```
以上就是Dockerfile文件的内容。

网站配置文件webapp.conf内容如下：
```
vim webapp.conf

# passenger 日志文件位置，如果不指定日志会在/var/log/nginx/error.log
passenger_debug_log_file /var/log/nginx/passenger.log;

server {
    #镜像内端口号
	listen 80;
    server_name www.example.com.cn;
	#网站public文件夹路径
    root /home/app/webapp/public;

    passenger_enabled on;
    passenger_user app;

    # 指定ruby文件夹版本:
    # For Ruby 2.1
    passenger_ruby /usr/bin/ruby2.1;

    # 是否部署为development模式，日志更详细，出现错误可以开启
    #passenger_app_env development;
}

```

## 3 创建镜像
进入刚刚创建的目录
```
cd myweb
```
docker build创建镜像
```
docker build -t webapp:1.0 .
```
此时需要等待一段时间，docker会将Dcokerfile中的每步操作增量封装镜像，直到最终完成，如有异常会立即停止执行并输出错误结果。

创建成功后，使用docker images查看镜像
```
docker images

REPOSITORY                           TAG                 IMAGE ID            CREATED              SIZE
webapp                               1.0                 3b6d00186d7e        3 hours ago          1.196 GB
docker.io/phusion/passenger-ruby21   0.9.27              d446db1d0dc8        4 weeks ago          684 MB
```
可以看到，镜像已经成功创建。

## 4 运行生成的镜像
```
docker run --name myweb --restart=always\
-p 80:80 \
-v /home/docker/data/openlab/log:/var/log \
-d \
webapp:1.0 

# -v存储映射日志，便于收集容器日志，需要关闭SElinux
# -p 开启宿主机的80端口映射
# -d 保持在后台运行
# --name myweb 指定docker运行别名
# --restart=always 容器异常退出时会自动重启
```

将docker加入开机启动
```
systemctl enable docker
```

进入镜像，可以进行各项操作
```
docker exec -it myweb bash
```

## 5 镜像的导出与导入

对于不能联网的机器，是不能使用docker pull从网上直接拉镜像的，另外，如果每次都从Dockerfile构建镜像的话也比较耗时间，这个时候就需要使用容器的导出与导入功能了。

### 5.1 镜像的导出
查看当前可用镜像
```
docker images

REPOSITORY                           TAG                 IMAGE ID            CREATED              SIZE
webapp                               1.0                 3b6d00186d7e        3 hours ago          1.196 GB
```
导出镜像
```
docker save -o /opt/webapp.tar webapp:1.0
#/opt/webapp.tar 导出镜像的位置及名称
#webapp:1.0 导出镜像的REPOSITORY与TAG
```
我们查看/opt下文件列表，可以看到导出成功
```
ll /opt

-rw-------.  1 root root 1237112320 Dec 26 14:24 webapp.tar
```

### 5.2 镜像的导入
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
docker load -i /opt/webapp.tar
```
再次使用docker images，可以看到镜像已经导入成功，可以在当前未联网机器上使用。

使用Docker之后，原本每次都需要1天左右才能部署好的Rails生产环境，现在需要数分钟即可拷贝部署完成，大大提高了迁移和部署效率。

## 6 关于rails网站部署的进一步探讨

1. 一般网站均存在静态文件，在多台机器上部署时需要进行进行同步，使用docker后，需要将静态文件映射到本地存储后再进行双向同步。
1. 本文讨论的仍然是传统的Rails网站部署，使用docker还可以部署编排可自动发现的网站集群系统。


# 参考资料
1. phusion/passenger-docker，https://github.com/phusion/passenger-docker
1. Deploying a Ruby app on a Linux/Unix production server，https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/ownserver/nginx/oss/xenial/deploy_app.html
1. Configuration reference for Passenger + Nginx，https://www.phusionpassenger.com/library/config/nginx/reference/#passenger_friendly_error_pages

# 如有疑问，欢迎交流