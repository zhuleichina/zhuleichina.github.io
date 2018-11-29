---
layout: post
title: 基于kendogrid的zabbix监控数据表格化展示
category: 工具
tags: [kendogrid, zabbix, nginx]
description: 本文将介绍基于kendogrid的zabbix监控数据表格化展示。
---

# 基于kendogrid的zabbix监控数据表格化展示

对于项目固定的zabbix监控数据来说，我们可以通过grafana进行图形化，表格化界面进行展示，但是对于zabbix自动发现的数据来说，因其会根据自动发现的项目动态变化，不适宜做固定项目的展示，因此需要一种展示zabbix自动发现数据的方法。

本文将介绍使用kendogrid表格化展示之前的基于zabbix自动发现采集的openstack虚机监控数据。

## 1 整体方案设计

我们可以先获取所有的虚机列表，然后根据虚机NAME，去读取zabbix对应的监控数据，因为虚机数目可能很多（800+），读取时间可能会比较长，直接做成页面刷新的时候再读取数据，可能会超时，因此就需要数据读取与数据展示的分离。

本文采用pyzabbix，即zabbix api in python读取自动发现数据，并保存为json格式，使用kendogrid读取json数据，使用nginx作为服务器进行页面展示。

## 2 使用pyzabbix获取监控数据

### 2.1 zabbix自动发现数据列表

Item name | Item key | 备注 
- | :-: | -: 
{#VMNAME} cpu_util | openstack.ceilometer[cpu_util,{#VMID}]| 
{#VMNAME} network.incoming.packets 1d | network_incoming_packets_1d_[{#VMID}] | 
{#VMNAME} network.incoming.packets 1h | openstack.ceilometer[network.incoming.packets,{#VMID}] | 
{#VMNAME} network.incoming.packets 1w | network_incoming_packets_1w_[{#VMID}]| 
{#VMNAME} network.outgoing.packets 1d | network_outgoing_packets_1d_[{#VMID}] | 
{#VMNAME} network.outgoing.packets 1h | openstack.ceilometer[network.outgoing.packets,{#VMID}] | 
{#VMNAME} network.outgoing.packets 1w | network_outgoing_packets_1w_[{#VMID}]| 
{#VMNAME} status | openstack.ceilometer[status,{#VMID}]| 
{#VMNAME} zone | openstack.ceilometer["OS-EXT-AZ:availability_zone",{#VMID}]| 

其中 {#VMNAME} 为虚机监控虚机的name, 我们可以据此查询对应虚机的数据。

### 2.2 获取虚机 name 列表

我们可以使用zabbix增加一个监控项来获取虚机的name列表

Item name | Item key | 备注 
- | :-: | -: 
Openstack VM name list | openstack.vm.name.list | 虚机name列表

对应的python实现如下：

```
def hyper_name_list():
    hypervisors = nova_client.hypervisors.list(False)
    name = []
    for hypervisor in hypervisors:
        name.append(hypervisor.hypervisor_hostname)
    print ",".join(name)
```

### 2.3 pyzabbix的安装与使用

#### 2.3.1 pyzabbix安装
```
pip install pyzabbix
```

#### 2.3.2 获取zabbix监控数据

```
from pyzabbix import ZabbixAPI
import json

# http://11.11.111.111/zabbix/ 为zabbix登录页面地址
zapi = ZabbixAPI("http://11.11.111.111/zabbix/")

# username为登录用户名，password为对应的密码
zapi.login("username ", "password")

# get_host_id 为获取host对应的id，env参数为易于理解的host姓名。
def get_host_id(env):
    if env=="tecs3":
        return 10106
    elif env=="tecs6":
        return 10169

# get_vm_data 为获取虚机数据并存储为json格式的主函数
def get_vm_data(env):
    filename = "/var/www/data/json/" + env + "_vm.json"
    hostid = get_host_id(env)

    vm_lists =zapi.item.get(output="extend",hostids=hostid,search={'name':'Openstack VM name list'})[0]["lastvalue"]
    vm_lists = vm_lists.split(',')
    print (vm_lists)
    data =[]
    for vm_name in vm_lists:
        print (vm_name)
        query_zone = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'zone')})[0]["lastvalue"]
        query_status = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'status')})[0]["lastvalue"]
        query_cpu_util = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'cpu_util')})[0]["lastvalue"]
        query_inconming1d = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'network.incoming.packets 1d')})[0]["lastvalue"]
        query_inconming1w = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'network.incoming.packets 1w')})[0]["lastvalue"]
        query_outgoing1d = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'network.outgoing.packets 1d')})[0]["lastvalue"]
        query_outgoing1w = zapi.item.get(output="extend",hostids=hostid,search={'name':(vm_name + ' ' + 'network.outgoing.packets 1w')})[0]["lastvalue"]

        data.append( {"name":vm_name,
                    "zone":query_zone,
                    "status":query_status, 
                    "cpu_util":query_cpu_util, 
                    "inconming1d":(float(query_inconming1d)), 
                    "inconming1w":(float(query_inconming1w)), 
                    "outgoing1d":(float(query_outgoing1d)), 
                    "outgoing1w":(float(query_outgoing1w))
                    } )
    
    # 对于获取到的数据，根据流量信息进行排序
    data.sort(key = lambda x: x["outgoing1w"])
    print(json.dumps(data, indent=2, encoding="utf-8"))
    
    #保存为json文件
    with open(filename, 'w') as f:
        f.write(json.dumps(data, indent=2, encoding="utf-8"))

调用如上函数，获取数据
get_vm_data("tecs3")
get_vm_data("tecs6")
```

执行之上的程序后，我们就可以获取如下格式的json数据
```
[
  {
    "status": "ERROR", 
    "cpu_util": "-1.0000", 
    "outgoing1d": 0.0, 
    "inconming1d": 0.0, 
    "name": "PFU_46/3/0", 
    "zone": "", 
    "inconming1w": 0.0, 
    "outgoing1w": 0.0
  },
  ……………………………………
```
我们可以根据源数据1小时刷新一次，将该脚本加入定时任务，这样就可以获取实时数据了。

## 3 使用kendogrid做表格化展示

对于上述脚本来说，我们保存json数据的路径为：/var/www/data/json/tecs3_vm.json

我们可以使用nginx,来对该数据进行部署，这样使用浏览器即可访问：

### nginx的安装与配置

安装与开机启动

```
yum install nginx
systemctl enable nginx
nginx
```

配置：
```
server
{
    listen 1001;
    server_name localhost;
    root  /var/www/data/;


    location / {
       # root   /var/www/data/;
       # index  index.html index.htm index.php;
       # try_files $uri $uri/ /index.php;
        alias /var/www/data/;
    }
}
```
这样当我们访问主机 ip:1001/json/tecs3_vm.json 时，即可获取对应的json数据

### kendogrid的表格展示

如下，保存为html文件，部署与 /var/www/data/ 内即可使用浏览器访问。

```
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta http-equiv="refresh" content="300">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    
    <link href="https://magicbox.bk.tencent.com/static_api/v3/assets/bootstrap-3.3.4/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://magicbox.bk.tencent.com/static_api/v3/assets/kendoui-2015.2.624/styles/kendo.common.min.css" rel="stylesheet">
    <link href="https://magicbox.bk.tencent.com/static_api/v3/assets/kendoui-2015.2.624/styles/kendo.default.min.css" rel="stylesheet">
    <link href="https://magicbox.bk.tencent.com/static_api/v3/assets/fontawesome/css/font-awesome.css" rel="stylesheet">
    <link href="https://magicbox.bk.tencent.com/static_api/v3/bk/css/bk.css" rel="stylesheet">


    <script src="https://magicbox.bk.tencent.com/static_api/v3/assets/js/jquery-1.10.2.min.js"></script>
    <script src="https://magicbox.bk.tencent.com/static_api/v3/assets/kendoui-2015.2.624/js/kendo.all.min.js"></script>
</head>


<body>
    <p class="magic-desc"><span class="strong">带排序搜索表格</span>，可排序、搜索</p>
    <div id="table3" class="mb30 demo"></div>
    <script>
        $(document).ready(function () {
            $('#table3').kendoGrid({
                pageable: false, //隐藏分页
                sortable: { //可排序
                    mode: "single",
                    allowUnsort: false
                },
                selectable: "multiple cell",
                allowCopy: true, //可复制
                dataSource: {
                    transport: {
                        read: {
                            url: "json/tecs3_vm.json",
                            dataType: "json"
                        }
                    }
                },
                toolbar: [
                    {
                        template:  '<div class="clearfix pt5 pb5 pl5">'+
                                    '<form class="form-inline king-search-form king-no-bg  pull-right">'+
                                    '  <div class="form-group">'+
                                    '      <label>搜索：</label>'+
                                    '      <div class="input-group">'+
                                    '          <input type="text" class="keyword form-control" placeholder="请输入虚机名称">'+
                                    '      </div>'+
                                    '      <div class="input-group">'+
                                    '          <input type="text" class="zonekeyword form-control" placeholder="请输入域名">'+
                                    '      </div>'+
                                    '  </div>'+
                                    '</form>'+
                                    '</div>'
                    }
                ],
                columns : [
                    { //第一列为自增序号显示
                        field: "rowNumber",
                        title: "序号",
                        width: 45,
                        template: "<span class='row-number'></span>"
                    },
                    {title : '虚机名称', field:"name", width:250},           
                    {title : '所属域', field:"zone"},           
                    {title : '虚机状态', field:"status"},
                    {
                        title : 'cpu使用率', 
                        field:"cpu_util",
                        template: function(ratio) { 
                            return kendo.htmlEncode(ratio.cpu_util) + "%";
                        }
                    },
                    {title : '入口流量--天', field:"inconming1d"},
                    {title : '入口流量--周', field:"inconming1w"},
                    {title : '出口流量--天', field:"outgoing1d"},
                    {title : '出口流量--周', field:"outgoing1w"}    
                ],
                dataBound: function () {
                    var rows = this.items();
                    $(rows).each(function () {
                        var index = $(this).index() + 1;
                        var rowLabel = $(this).find(".row-number");
                        $(rowLabel).html(index);
                    });
                },
            });
            var grid1 = $('#table3').data('kendoGrid'); //获取kendoUI grid对象
            $('#table3').find('.keyword').on('keyup',function(){
                var keyword = $(this).val();
                grid1.dataSource.filter({
                    field: 'name', //设置搜索字段
                    operator: 'contains', //包含关键字
                    value: keyword
                });
            });

            $('#table3').find('.zonekeyword').on('keyup',function(){
                var zonekeyword = $(this).val();
                grid1.dataSource.filter({
                    field: 'zone', //设置搜索字段
                    operator: 'contains', //包含关键字
                    value: zonekeyword
                });
            });
            //table4_demo1_js_end
        
        });
    </script>

</body>

</html>
```

其中dataSource配置了json数据的路径，因部署与同一路径下，所有并未使用 ip:1001/json/tecs3_vm.json 
```
dataSource: {
    transport: {
        read: {
            url: "json/tecs3_vm.json",
            dataType: "json"
        }
    }
},
```

如上，即完成了zabbix自动发现的虚机监控数据的 kendogrid 的表格化展示。

# 参考资料
1. lukecyca/pyzabbix, https://github.com/lukecyca/pyzabbix
2. 带排序搜索表格(Kendo UI), http://magicbox.bk.tencent.com/#detail/show?id=table4&isPro=0
3. Python 针对Json中某个关键字段进行排序, https://blog.csdn.net/vitaminc4/article/details/77994120

# 如有疑问，欢迎交流
