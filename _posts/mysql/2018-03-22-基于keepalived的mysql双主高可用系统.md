---
layout: post
title: 基于keepalived的mysql双主高可用系统
category: mysql
tags: [keepalived , mysql]
description: mysql单节点存储时，系统出现故障时服务不可用、不能及时恢复的问题，因此实际使用时，一般都会使用mysql双机方案，使用keepalived实现mysql双主是较常见的一种双机方案。
---

# 基于keepalived的mysql双主高可用系统

mysql单节点存储时，系统出现故障时服务不可用、不能及时恢复的问题，因此实际使用时，一般都会使用mysql双机方案，使用keepalived实现mysql双主是较常见的一种双机方案。该系统主要实现了以下功能：
1. 当其中一台机器mysql出现异常时，keepalived脚本会自动重启；
2. 重启失败后会降低优先级变为不可用机，由另外一台机器接管VIP，对外提供服务；
3. 当不可用机mysql进程恢复后，会提高优先级自动变为备用机；
4. 使用cron定时脚本监控keepalived进程，当keepalived进程异常退出时会自动重启服务；

本文将按照以下顺序介绍该系统的实现：
1. 环境搭建：网络规划及mysql和keepalived软件准备；
2. mysql双主复制环境搭建；
3. keepalived配置与测试；
4. 基于cron定时任务的keepalived进程监控；


## 1 环境搭建

### 1.1 网络规划

| 角色          | IP地址           
| -------------|:-------------:|
| mysql1       | 192.168.1.10  |
| mysql2       | 192.168.1.11  |
| VIP          | 192.168.1.12  |

### 1.2 软件准备

主要是在两台机器上安装mysql和keepalived软件。

#### 1.2.1 mysql的安装

a、下载
```
下载地址：http://mirrors.sohu.com/mysql/MySQL-5.6/ (说明：也可以去官网下，非常慢)
下载文件：mysql-5.6.36-linux-glibc2.5-x86_64.tar.gz
拷贝文件：使用Xftp工具，将安装文件拷贝到 /opt文件夹
```
b、解压
```
#解压
tar -zxvf mysql-5.6.36-linux-glibc2.5-x86_64.tar.gz
#复制解压后的mysql目录
cp -r mysql-5.6.36-linux-glibc2.5-x86_64 /usr/local/mysql
```
c、添加用户组和用户
```
#添加用户组
>groupadd mysql
#添加用户mysql 到用户组mysql
useradd -g mysql mysql
```
d、安装
```
cd /usr/local/mysql/
chown -R mysql:mysql ./
./scripts/mysql_install_db --user=mysql --datadir=/usr/local/mysql/data
cp support-files/mysql.server /etc/init.d/mysqld
chmod 755 /etc/init.d/mysqld
cp support-files/my-default.cnf /etc/my.cnf

#修改启动脚本
vi /etc/init.d/mysqld
 
#修改项：
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
 
#启动服务
service mysqld start
 
#测试连接
./mysql/bin/mysql -uroot
 
#加入环境变量，编辑 /etc/profile，这样可以在任何地方用mysql命令了
export PATH=$PATH:/usr/local/mysql//bin
```
e、mysqld服务开机启动
```
#添加可执行权限
chmod +x /etc/init.d/mysqld
#加入开机启动项
chkconfig --add mysqld 
#查看是否添加成功
chkconfig --list mysqld 
```
f、修改mysql配置文件/etc/my.cnf，指定.sock的位置
```
增加如下配置选项：
[mysqld]
# These are commonly set, remove the # and set as required.
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
socket = /var/lib/mysql/mysql.sock

sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES 

[mysql]
socket = /var/lib/mysql/mysql.sock

[mysqldump]
socket = /var/lib/mysql/mysql.sock

[mysqld_safe]
log-error=/var/log/mysql/mysql.log
```
g、其他配置选项
```
mysql目录下修改文件所有者：
chown -R root:root ./
chown -R mysql:mysql data

创建日志文件
cd /var/log
mkdir mysql
cd mysql
touch mysql.log
chmod -R 775 mysql.log
chown -R mysql:mysql mysql.log

创建sock文件夹
mkdir /var/lib/mysql/
chmod 777 /var/lib/mysql/
```
h、启动mysql
```
service mysqld start
```
i、修改密码
```
通过登录mysql系统，
# mysql -uroot -p
Enter password: 【输入原来的密码：默认为空】
mysql>use mysql;
mysql> update user set password=password("123456") where user='root';
mysql> flush privileges;
mysql> exit;   
```
至此，mysql安装启动完成。

#### 1.2.2 keepalived的安装
使用yum安装
```
yum install keepalived
```
若不能使用yum安装，本地安装方法如下：
```
cd /opt
wget http://www.keepalived.org/software/keepalived-1.3.9.tar.gz
tar -zxvf keepalived-1.3.9.tar.gz
cd keepalived-1.3.9
./configure --prefix=/usr/local/keepalived
make && make install
cp /usr/local/keepalived/etc/keepalived.conf /etc/keepalived/keepalived.conf
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived
```
一般的参考文档中提到了将keepalived键入init.d中可以使用system进行开启和关闭，但最新版本keepalived安装完成后，并没有该文件，使用时，开启和关闭方法如下：
```
#开启
keepalived
#关闭
pkill keepalived
#加入开机启动项，编辑rc.local，增加keepalived
vim /etc/rc.local
…………
keepalived
```
keepalived日志文件为 /var/log/messages


## 2 mysql双主复制环境搭建

### 2.1 修改mysql配置文件

a、mysql1机器上
```
vim /etc/my.cnf
#在mysqld下增加如下内容
[mysqld]
log-bin=mysql-bin
server-id=1
log_slave_updates=1
```
b、mysql2机器上
```
vim /etc/my.cnf
#在mysqld下增加如下内容
[mysqld]
log-bin=mysql-bin
server-id=2
log_slave_updates=1
```

### 2.2 在mysql上对远程用户互相进行授权登录

a、mysql1机器上
```
#192.168.1.11为mysql2的IP地址

mysql -u root -pxxxxxx
mysql>use mysql;
mysql>GRANT REPLICATION SLAVE ON *.* TO 'root'@'192.168.1.11';
mysql>select host, user from user;
mysql>flush privileges;
mysql> exit;
```
b、mysql2机器上
```
#192.168.1.10为mysql1的IP地址

mysql -u root -pxxxxxx
mysql>use mysql;
mysql>GRANT REPLICATION SLAVE ON *.* TO 'root'@'192.168.1.10';
mysql>select host, user from user;
mysql>flush privileges;
mysql>exit;
```

### 2.3 设置mysql主从复制环境
a、mysql1机器
```
#在mysql2上，查询相关status
mysql>show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |     2453 |              |                  |
+------------------+----------+--------------+------------------+
#记录File和Position，然后在mysql1上进行设置
mysql>CHANGE MASTER TO 
        MASTER_HOST='192.168.1.11',
        MASTER_USER='root',
        MASTER_PASSWORD='123456',
        MASTER_LOG_FILE='mysql-bin.000001',
        MASTER_LOG_POS=2453;
```
b、mysql2机器
```
#在mysql1上，查询相关status
mysql>show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |     3155 |              |                  |
+------------------+----------+--------------+------------------+
#记录File和Position，然后在mysql2上进行设置
mysql>CHANGE MASTER TO 
        MASTER_HOST='192.168.1.10',
        MASTER_USER='root',
        MASTER_PASSWORD='123456',
        MASTER_LOG_FILE='mysql-bin.000001',
        MASTER_LOG_POS=3155;
```

### 2.4 验证设置
```
mysql>show slave status\G
#若结果中包含如下，即认为设置成功：
Slave_IO_Running: Yes
Slave_SQL_Running: Yes

#在mysql1上创建table
CREATE TABLE test1111(id INT NOT NULL AUTO_INCREMENT,PRIMARY KEY(id));
#在mysql2上查看tables发现已经同步过去了
mysql>show tables;
+----------------------------------+
| Tables_in_zte_openlab_production |
+----------------------------------+
| test1111                         |
+----------------------------------+
```

## 3 keepalived配置与测试

### 3.1 keepalived配置

配置文件为：/etc/keepalived/keepalived.conf
```
! Configuration File for keepalived

global_defs {
   notification_email {
     #acassen@firewall.loc
     #failover@firewall.loc
     #sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_BACKUP
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script chk_mysql {
    script "/etc/keepalived/check_mysql.sh"   # mysql 服务检测并试图重启
    interval 5                    # 每5s检查一次
    weight -2                    # 检测失败（脚本返回非0）则优先级减少2个值
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth3   #eth3为keepalived的心跳网口，可以和业务同网口  
    virtual_router_id 51
    priority 100  #mysql2上，优先级也设置为100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.1.12/24 dev eth0  #虚拟IP绑定的业务口
    }
	
	track_script {  
        chk_mysql
    }
}

```
vrrp_script模块中指定的脚本 
/etc/keepalived/check_mysql.sh
其内容为：
```
#!/bin/bash
mysql_counter=$(ps -C mysqld --no-heading|wc -l)

if [ "${mysql_counter}" = "0" ]; then
    service mysqld start
	sleep 2
fi

mysql_counter=$(ps -C mysqld --no-heading|wc -l)

if [ "${mysql_counter}" = "0"]; then
    exit 1
else
	exit 0
fi

需要可执行权限：
chmod +x /etc/keepalived/check_mysql.sh
```
即检测mysql进程若退出，则重启，重启成功返回值为0，优先级priority不变，若重启失败则返回值为1，优先级priority-2变为98，由备用节点keepalived接管VIP对外提供服务。

### 3.2 keepalived相关脚本测试
```
#使用service关闭mysql进程，查看进程是否会重启
[root@localhost keepalived]# service mysqld stop
Shutting down MySQL....                                    [  OK  ]
[root@localhost keepalived]# ps aux | grep mysqld
root     116814  0.1  0.0  11764  1580 ?        S    18:44   0:00 /bin/sh 
mysql    117046  0.7  2.8 671136 109632 ?       Sl   18:44   0:00 /usr/local/mysql/bin/mysqld --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --plugin-dir=/usr/local/mysql/lib/plugin --user=mysql --log-error=/var/log/mariadb/mariadb.log --pid-file=/usr/local/mysql/data/localhost.localdomain.pid --socket=/var/lib/mysql/mysql.sock --port=3306
root     117102  0.0  0.0 112644   952 pts/4    S+   18:44   0:00 grep --color=auto mysqld

#可以看到手动停止mysql进程后，脚本执行成功执行重启脚本，且未发生VIP漂移。

#修改脚本内容，将重启mysql语句注释：
#service mysqld start
#再次停止mysql进程，此时messages日志如下：
Keepalived_vrrp[64509]: /etc/keepalived/check_mysql.sh exited with status 1
Keepalived_vrrp[64509]: VRRP_Script(chk_mysql) failed
Keepalived_vrrp[64509]: VRRP_Instance(VI_1) Changing effective priority from 100 to 98
Keepalived_vrrp[64509]: VRRP_Instance(VI_1) Received advert with higher priority 100, ours 98
Keepalived_vrrp[64509]: VRRP_Instance(VI_1) Entering BACKUP STATE
#即脚本执行失败，降低优先级，发生VIP漂移，能够继续对外提供服务
#手动重启mysql进程，查看日志，优先级升高，但不会发生漂移
Keepalived_vrrp[64509]: /etc/keepalived/check_mysql.sh exited with status 1
Keepalived_vrrp[64509]: VRRP_Script(chk_mysql) succeeded
Keepalived_vrrp[64509]: VRRP_Instance(VI_1) Changing effective priority from 98 to 100
```
测试结果达到预期要求

### 3.3 keepalived，VIP无法ping通的问题
上述配置完成后，能够正常切换，但VIP始终无法ping通，后发现手动执行增加IP的指令可以ping通
```
ip addr add 192.168.1.12/24 dev eth0:0
```
于是脚本修改为：
```
#删除如下内容：
    virtual_ipaddress {
        192.168.1.12/24 dev eth0  #虚拟IP绑定的业务口
    }

#替换为如下配置：
    notify_master /etc/keepalived/vipup.sh
    notify_backup /etc/keepalived/vipdown.sh
    notify_fault /etc/keepalived/vipdown.sh
    notify_stop /etc/keepalived/vipdown.sh
#相关脚本配置
vim /etc/keepalived/vipup.sh
#!/bin/sh
ip addr add 192.168.1.12/24 dev eth0:0
arping  -c 5 -U -I eth0 192.168.1.12
#其中arp指令为对外宣告新增加网卡的ARP信息
vim /etc/keepalived/vipdown.sh
#!/bin/sh
ip addr del 192.168.1.12/24 dev eth0:0

#均需要可执行权限：
chmod +x /etc/keepalived/vipup.sh
chmod +x /etc/keepalived/vipdown.sh
```
按如上配置进行修改后，VIP可以ping通，对外可以提供mysql服务

## 4 基于cron定时任务的keepalived进程监控

### 4.1 cron定时任务相关配置

经过上述配置后，能够保证mysql的高可用性，但对于keepalived来说，一旦keepalived进程异常退出，就会发生VIP漂移，且需要人工干预重启，于是想到能否定时检测keepalived进程，并进行自动重启；
```
#/var/spool/cron/root增加如下定时任务，即10s执行一次检测脚本
vim /var/spool/cron/root 
* * * * * sleep 10;/etc/keepalived/check_keepalived.sh
#脚本内容如下：
vim /etc/keepalived/check_keepalived.sh
#!/bin/bash
keepalived_counter=$(ps -C keepalived --no-heading|wc -l)
if [ "${keepalived_counter}" = "0" ]; then
    /usr/sbin/keepalived
fi
#需要可执行权限
chmod +x /etc/keepalived/check_keepalived.sh
```

### 4.2 进程退出测试
```
#手动退出keepalived进程，查看日志文件

Keepalived[116366]: Stopping
Keepalived_healthcheckers[116367]: Stopped
Keepalived_vrrp[116368]: VRRP_Instance(VI_1) sent 0 priority
avahi-daemon[1112]: Withdrawing address record for 192.168.1.12 on eth0.
Keepalived_vrrp[116368]: Stopped
Keepalived[116366]: Stopped Keepalived v1.3.9 (10/21,2017)
Keepalived[125696]: Starting Keepalived v1.3.9 (10/21,2017)
Keepalived[125696]: Opening file '/etc/keepalived/keepalived.conf'.
Keepalived[125697]: Starting Healthcheck child process, pid=125698
Keepalived[125697]: Starting VRRP child process, pid=125699
Keepalived_healthcheckers[125698]: Opening file '/etc/keepalived/keepalived.conf'.
Keepalived_vrrp[125699]: Registering Kernel netlink reflector
Keepalived_vrrp[125699]: Registering Kernel netlink command channel
Keepalived_vrrp[125699]: Registering gratuitous ARP shared channel
Keepalived_vrrp[125699]: Opening file '/etc/keepalived/keepalived.conf'.
Keepalived_vrrp[125699]: WARNING - default user 'keepalived_script' for script execution does not exist - please create.
Keepalived_vrrp[125699]: SECURITY VIOLATION - scripts are being executed but script_security not enabled.
Keepalived_vrrp[125699]: Using LinkWatch kernel netlink reflector...
Keepalived_vrrp[125699]: VRRP_Instance(VI_1) Entering BACKUP STATE
Keepalived_vrrp[125699]: VRRP_Script(chk_mysql) succeeded

#可以看到keepalived进程异常退出后，又自动重启的过程，测试完成
```

## 参考资料
1. 利用keepalived构建高可用MySQL-HA，http://blog.csdn.net/catoop/article/details/41285001
2. 基于keepalived搭建MySQL的高可用集群，https://www.cnblogs.com/ivictor/p/5522383.html
3. Linux下安装配置使用 Keepalived,http://blog.csdn.net/conquer0715/article/details/47955553
4. Linux VIP(虚拟IP)配置后,无法ping通的问题处理,http://blog.csdn.net/zhang_shufeng/article/details/37930405
5. Keepalived无法绑定VIP故障排查经历,http://www.linuxidc.com/Linux/2015-03/114981.htm
6. linux crontab & 每隔10秒执行一次，https://www.cnblogs.com/juandx/archive/2015/11/24/4992465.html

## 若有疑问，欢迎交流