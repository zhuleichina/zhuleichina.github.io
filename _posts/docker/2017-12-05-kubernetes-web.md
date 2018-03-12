---
layout: post
title: kubernetes实践-使用kubernets部署web应用
category: docker
tags: [docker]
description: 使用kubernets部署web应用
---

# kubernetes实践-使用kubernets部署web应用

在之前的文章：
docker实践--基于docker的rails网站部署，https://dev.zte.com.cn/topic/#/44381

我们使用Dockerfile文件构建了rails网站镜像并成功运行，本节我们将学习使用kubernets集群部署该镜像。

## 1 kubernetes可操作对象介绍

### 1.1 pod
pod是Kubernetes最基本的部署调度单元，一个pod可以看成是由若干个容器组成的容器组。

### 1.2 replicationController
replicationController是pod的复制抽象，用于解决pod的扩容缩容问题。通常，分布式应用为了性能或高可用性的考虑，需要复制多份资源，并且根据负载情况动态伸缩。通过replicationController，我们可以指定一个应用需要几份复制，Kubernetes将为每份复制创建一个pod，并且保证实际运行pod数量总是与该复制数量相等。

### 1.3 service
是pod的路由代理抽象，用于解决pod之间的服务发现问题。因为pod的运行状态可动态变化，所以访问端不能以写死IP的方式去访问该pod提供的服务。service的引入旨在保证pod的动态变化对访问端透明，访问端只需要知道service的地址，由service来提供代理。

## 2 环境准备

### 2.1 使用kubeadm部署kubernetes集群
按照文档，使用kubeadm部署kubernetes集群,https://dev.zte.com.cn/topic/#/44779 
部署含有两个node节点的kubernetes集群。

### 2.2 rails网站镜像
将网站镜像：webapp:1.0，导入到两台node节点中。

一般情况下会设定私有镜像库的地址，使用镜像库中存在的镜像即可，不需要单独下载镜像。

## 3 web应用部署

### 3.1 部署pod及复制器

在kubernetes集群的master节点，在工作目录，新建webapp-rc.yaml文件，文件内容如下：
```
vim webapp-rc.yaml

apiVersion: v1 
kind: ReplicationController 
metadata: 
  name: web-controller 
spec: 
  #两个副本
  replicas: 2 
  #这个ReplicationController中包含如下pod
  selector: 
    name: web 
  template: 
    metadata: 
      labels: 
        name: web 
    spec: 
      containers: 
        - name: web 
          image: webapp:1.0 
          ports: 
            - containerPort: 80

```

使用kubectl应用该模版，创建replicationcontroller
```
kubectl create -f webapp-rc.yaml

replicationcontroller "web-controller" created
```

使用kubectl get pods查询是否创建成功
```
kubectl get pods

NAME                   READY     STATUS    RESTARTS   AGE
web-controller-89zck   1/1       Running   0          42s
web-controller-m5gcz   1/1       Running   0          42s
```
如果在node上没有提前准备镜像文件webapp:1.0，会提示拉取镜像失败。

我们可以使用describe 命令查看pod的相关信息：
```
kubectl describe pod web-controller-89zck

Name:		web-controller-89zck
Namespace:	default
Node:		slave1/192.168.1.12
Start Time:	Wed, 10 Jan 2018 15:30:01 +0800
Labels:		name=web
Annotations:	kubernetes.io/created-by={"kind":"SerializedReference","apiVersion":"v1","reference":{"kind":"ReplicationController","namespace":"default","name":"web-controller","uid":"1183922f-f5d8-11e7-95f4-fa0000...
Status:		Running
IP:		10.244.1.6
Created By:	ReplicationController/web-controller
Controlled By:	ReplicationController/web-controller
Containers:
  web:
    Container ID:	docker://e8e77ef37b2ca249033b66f76083562ae8a90d4b43b96f4d9c5bc0eb4145f746
    Image:		webapp:1.0
    Image ID:		docker://sha256:f0076ea088e325d2b9e6e1c9c47ebf71182b3e793c277e62b13726bf4836241a
    Port:		80/TCP
    State:		Running
      Started:		Wed, 10 Jan 2018 15:30:05 +0800
    Ready:		True
    Restart Count:	0
    Environment:	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-c8m64 (ro)
Conditions:
  Type		Status
  Initialized 	True 
  Ready 	True 
  PodScheduled 	True 
Volumes:
  default-token-c8m64:
    Type:	Secret (a volume populated by a Secret)
    SecretName:	default-token-c8m64
    Optional:	false
QoS Class:	BestEffort
Node-Selectors:	<none>
Tolerations:	node.alpha.kubernetes.io/notReady:NoExecute for 300s
		node.alpha.kubernetes.io/unreachable:NoExecute for 300s
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath		Type		Reason			Message
  ---------	--------	-----	----			-------------		--------	------			-------
  6m		6m		1	default-scheduler				Normal		Scheduled		Successfully assigned web-controller-89zck to slave1
  6m		6m		1	kubelet, slave1					Normal		SuccessfulMountVolume	MountVolume.SetUp succeeded for volume "default-token-c8m64" 
  6m		6m		1	kubelet, slave1		spec.containers{web}	Normal		Pulled			Container image "webapp:1.0" already present on machine
  6m		6m		1	kubelet, slave1		spec.containers{web}	Normal		Created			Created container
  6m		6m		1	kubelet, slave1		spec.containers{web}	Normal		Started			Started container

#Node:slave1/192.168.1.12,为该pod的宿主机
#Containers展示了当前pod内的镜像信息
#Events展示了pod从调度到镜像创建的全过程事件
```

我们还可以尝试删除该pod：
```
kubectl delete pod web-controller-89zck

pod "web-controller-89zck" deleted
```
再次使用get pods查看pod状态
```
kubectl get pods

NAME                   READY     STATUS    RESTARTS   AGE
web-controller-6v5h4   1/1       Running   0          1m
web-controller-m5gcz   1/1       Running   0          30m
```
可以看到，原本的web-controller-89zck已经被删除，但kubernetes集群自动新建了一个pod，web-controller-6v5h4且正常运行，而这一切都是自动进行的。

pod及复制器已经部署成功，但是对外还不能提供访问，下面我们部署service提供访问接口。

### 3.2 部署service

service通常分为三种类型，分别为ClusterIP、NodePort和LoadBalancer。

ClusterIP是最基本的类型，在默认情况下只能在集群内部进行访问；另外两种则可以从外部进行访问。

如果将service的类型设为NodePort，那么系统会从service-node-port-range范围内为其分配一个端口，且在每个工作节点上都打开该端口，使得访问该端口即可访问该service对应的pod。

类型为LoadBalancer的service较为特殊，它需要云服务提供商提供支持而不是由kubernetes集群维护。

我们主要部署前两种类型的service。

#### 3.2.1 ClusterIP类型的service

在工作目录创建webapp-service-clusterip.yaml文件，配置如下：
```
vim webapp-service-clusterip.yaml 

apiVersion: v1 
kind: Service 
metadata: 
  name: web-service-clusterip 
spec: 
  ports: 
    - port: 8001 
      targetPort: 80 
      protocol: TCP 
  selector: 
    name: web
```
上述配置文件会将pod中的80端口影射为由系统分配的clusterIP的8001端口，对集群内部提供访问。

使用kubectl应用该模版，创建service
```
kubectl create -f webapp-service-clusterip.yaml 

service "web-service-clusterip" created
```

使用get service查看创建的service
```
kubectl get service
NAME                    CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes              10.96.0.1       <none>        443/TCP    4d
web-service-clusterip   10.108.231.24   <none>        8001/TCP   40s
```
上面的输出说明系统分配了clusterIP为10.108.231.24，PORT为8001，我们可以在集群中的任何一个节点来验证该service的可用性：
```
curl -s 10.108.231.24:8001

#会输出网站首页的源码
```

这样我们就成功创建了ClusterIP类型的service，且验证了其在集群内部可以提供服务。

#### 3.2.2 NodePort类型的service
在工作目录创建webapp-service-nodeport.yaml文件，配置如下：
```
vim webapp-service-nodeport.yaml 

apiVersion: v1 
kind: Service 
metadata: 
  name: web-service-nodeport 
spec: 
  ports: 
    - port: 8000
      targetPort: 80 
      protocol: TCP 
  type: NodePort
  selector: 
    name: web
```
我们指定了type: NodePort类型的service，使用kubectl应用该模版，创建service
```
kubectl create -f webapp-service-nodeport.yaml 

service "web-service-nodeport" created
```
使用get service查看创建的service
```
kubectl get service

NAME                    CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes              10.96.0.1        <none>        443/TCP          4d
web-service-clusterip   10.108.231.24    <none>        8001/TCP         13m
web-service-nodeport    10.104.165.219   <nodes>       8000:32566/TCP   1m
```
service NAME为web-service-nodeport创建成功，使用describe查询其详细信息：
```
kubectl describe service web-service-nodeport

Name:			web-service-nodeport
Namespace:		default
Labels:			<none>
Annotations:		<none>
Selector:		name=web
Type:			NodePort
IP:			10.104.165.219
Port:			<unset>	8000/TCP
NodePort:		<unset>	32566/TCP
Endpoints:		10.244.1.7:80,10.244.2.5:80
Session Affinity:	None
Events:			<none>
```
说明系统分配了NodePort为32566的端口对外提供访问。

我们可以在集群外使用如下地址进行访问：
```
http://192.168.1.11:32566/

#192.168.1.11为master节点的IP，也可以通过node节点的IP+NodePort进行访问
```
这样我们就成功创建了NodePort类型的service，且验证了其可以在集群外部可以访问。


# 参考资料
1. Kubernetes初探，http://blog.csdn.net/zhangjun2915/article/details/40598151
2. CentOS7.0上部署kubernetes集群 + 简单应用示例，http://blog.csdn.net/qq_21398167/article/details/52815901

# 如有疑问，欢迎交流