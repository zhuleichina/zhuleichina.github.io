---
layout: post
title: 使用kubeadm部署kubernetes集群
category: docker
tags: [docker]
description: 使用kubeadm部署kubernetes集群
---

# kubernetes实践-使用kubeadm部署kubernetes集群

通过docker，我们可以在单个主机上快速部署各个应用，但是实际的生产环境里，不会单单存在一台主机，这就需要用到docker集群管理工具了，本文将简单介绍使用docker集群管理工具kubernetes进行集群部署。

## 1 环境规划与准备

本次搭建使用了三台主机，其环境信息如下：
| 节点功能     |  主机名     |       IP     |
| ------------|:-----------:|-------------:|
| master      |   master    |192.168.1.11  |
| slave1      |   slave1    |192.168.1.12  |
| slave2      |   slave2    |192.168.1.13  |

在三台主机的/etc/hosts文件中添加以下内容
```
vim /etc/hosts
#添加以下信息
192.168.1.11 master
192.168.1.12 slave1
192.168.1.13 slave2
```

### 关闭swap
```
swapoff -a
```
再把/etc/fstab文件中带有swap的行注释。

### 关闭SELinux
```
setenforce 0
vim /etc/sysconfig/selinux
#修改SELINUX属性
SELINUX=disabled
```

### 设置iptables
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF

sysctl --system
```

### 安装socat等工具
```
yum install -y ebtables socat
```

## 2 kubernetes安装

### 2.1 安装docker

官方推荐安装docker版本为1.12
```
#yum安装docker
yum install -y docker
#设置docker开机启动
systemctl enable docker
#启动docker
systemctl start docker
```
验证docker版本
```
docker --version
#以下为输出的版本信息
Docker version 1.12.6, build 85d7426/1.12.6
```

### 2.2 安装kubectl、kubelet、kubeadm

#### 2.2.1 添加yum源
常用yum源均没有这几个安装包，需要添加专门的yum源
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF

#官方文档中的yum源为google，国内无法使用
```
#### 2.2.2 安装kubectl、kubelet、kubeadm
```
yum install -y kubelet kubeadm kubectl
```
#### 2.2.3 启动kubelet
```
#设置开机启动kubelet
systemctl enable kubelet
#启动kubelet
systemctl start kubelet
```
查询kubelet的状态
```
systemctl status kubelet
```
初次安装的情况下kubelet应未启动成功，我们会按下面的步骤初始化集群后会自动启动的。

## 3 kubernetes集群配置

### 3.1 master节点配置

#### 3.1.1 初始化master
根据官方文档进行初始化：
```
kubeadm init --apiserver-advertise-address 192.168.1.11 --pod-network-cidr 10.244.0.0/16

#--apiserver-advertise-address 192.168.1.11为master节点IP，部分文档也指定为0.0.0.0
#--pod-network-cidr 10.244.0.0/16为pod网络cidr
```
出现如下错误：
```
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
unable to get URL "https://storage.googleapis.com/kubernetes-release/release/stable-1.7.txt": Get https://storage.googleapis.com/kubernetes-release/release/stable-1.7.txt: net/http: TLS handshake timeout
```
需要指定kubernetes-version。

首先查询版本
```
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.5",…………
```
版本为1.7.5，然后启动参数中加入版本：
```
kubeadm init --apiserver-advertise-address 192.168.1.11 --pod-network-cidr 10.244.0.0/16 --kubernetes-version=v1.7.5
```

#### 3.1.2 master节点依赖镜像

执行过程中会卡在如下步骤：
```
[apiclient] Created API client, waiting for the control plane to become ready
```
因为kubenetes初始化启动会依赖某些镜像，而这些镜像默认会到google下载，我们需要手动下载下来这些镜像后再进行初始化。

使用CTRL+C结束当前进程，然后到/etc/kubernetes/manifests/目录下查看各个yaml文件，还有其他需要的镜像文件，汇总后如下：
```
gcr.io/google_containers/etcd-amd64:3.0.17
gcr.io/google_containers/kube-apiserver-amd64:v1.7.5
gcr.io/google_containers/kube-controller-manager-amd64:v1.7.5
gcr.io/google_containers/kube-scheduler-amd64:v1.7.5
gcr.io/google_containers/pause-amd64:3.0
gcr.io/google_containers/kube-proxy-amd64:v1.7.5
quay.io/coreos/flannel
gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4
gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4


```
因直接下载这些google镜像，下载不下来，我们通过下载dockerHUB/阿里云上的镜像，然后更改tag。
```
#etcd-amd64:3.0.17
docker pull sylzd/etcd-amd64-3.0.17
docker tag docker.io/sylzd/etcd-amd64-3.0.17:latest gcr.io/google_containers/etcd-amd64:3.0.17

#kube-apiserver-amd64:v1.7.5
docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/kube-apiserver-amd64:v1.7.5
docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/kube-apiserver-amd64:v1.7.5 gcr.io/google_containers/kube-apiserver-amd64:v1.7.5

#kube-controller-manager-amd64:v1.7.5
docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/kube-controller-manager-amd64:v1.7.5
docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/kube-controller-manager-amd64:v1.7.5 gcr.io/google_containers/kube-controller-manager-amd64:v1.7.5

#kube-scheduler-amd64:v1.7.5
docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/kube-scheduler-amd64:v1.7.5
docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/kube-scheduler-amd64:v1.7.5 gcr.io/google_containers/kube-scheduler-amd64:v1.7.5

#pause-amd64:3.0
docker pull visenzek8s/pause-amd64:3.0
docker tag visenzek8s/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0

#kube-proxy-amd64:v1.7.5
docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.7.5
docker tag mirrorgooglecontainers/kube-proxy-amd64:v1.7.5 gcr.io/google_containers/kube-proxy-amd64:v1.7.5

#quay.io/coreos/flannel
docker pull quay.io/coreos/flannel

#k8s-dns-kube-dns-amd64:1.14.4
docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/k8s-dns-kube-dns-amd64:1.14.4
docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/k8s-dns-kube-dns-amd64:1.14.4 gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4

#k8s-dns-sidecar-amd64:1.14.4
docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/k8s-dns-sidecar-amd64:1.14.4
docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/k8s-dns-sidecar-amd64:1.14.4 gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4

#k8s-dns-dnsmasq-nanny-amd64:1.14.4
docker pull mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.4
docker tag mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.4 gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4
```

master节点初始化成功，结果如下：
```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 3f1db4.9f7ba7d52de40996 192.168.1.11:6443

```
需要记住kubeadm join --token这句，后面会用到

#### 3.1.3 kubectl配置
```
#对于非root用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#对于root用户
export KUBECONFIG=/etc/kubernetes/admin.conf
```

#### 3.1.4 pod network配置
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

安装完network之后，你可以通过kubectl get pods --all-namespaces来查看kube-dns是否在running来判断network是否安装成功。
```
kubectl get pods --all-namespaces

#运行正常的结果如下：
NAMESPACE     NAME                                            READY     STATUS    RESTARTS   AGE
kube-system   etcd-localhost.localdomain                      1/1       Running   0          1h
kube-system   kube-apiserver-localhost.localdomain            1/1       Running   0          1h
kube-system   kube-controller-manager-localhost.localdomain   1/1       Running   3          1h
kube-system   kube-dns-2425271678-27g6v                       3/3       Running   0          1h
kube-system   kube-flannel-ds-1mjq3                           1/1       Running   1          1h
kube-system   kube-proxy-mtjwb                                1/1       Running   0          1h
kube-system   kube-scheduler-localhost.localdomain            1/1       Running   0          1h
```
如果以上STATUS中存在不是Running的需要再进行解决。

由于安全原因，默认情况下pod不会被schedule到master节点上，可以通过下面命令解除这种限制：
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 3.2 slave节点配置

#### 3.2.1 slave节点依赖镜像
slave节点需要以下镜像：
```
gcr.io/google_containers/kube-proxy-amd64:v1.7.5
quay.io/coreos/flannel
gcr.io/google_containers/pause-amd64:3.0
```
在msater节点上导出镜像
```
docker save -o /opt/kube-pause.tar gcr.io/google_containers/pause-amd64:3.0
docker save -o /opt/kube-proxy.tar gcr.io/google_containers/kube-proxy-amd64:v1.7.5
docker save -o /opt/kube-flannel.tar quay.io/coreos/flannel
```
复制到slave主机/opt目录下，再导入即可：
```
docker load -i /opt/kube-flannel.tar
docker load -i /opt/kube-proxy.tar
docker load -i /opt/kube-pause.tar
```

#### 3.2.2 slave节点加入集群

在两个slave节点上执行：
```
kubeadm join --token 3f1db4.9f7ba7d52de40996 192.168.1.11:6443
```
执行成功标志：
```
Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```
在mster节点上执行kubectl get nodes查看是否成功：
```
kubectl get nodes
NAME      STATUS    AGE       VERSION
master    Ready     56m       v1.7.5
slave1    Ready     1m        v1.7.5
slave2    Ready     1m        v1.7.5
```

可以看到，kubernetes集群已经部署成功，可以使用了。

# 参考资料
1. Installing kubeadm，https://kubernetes.io/docs/setup/independent/install-kubeadm/
2. 使用kubeadm在Red Hat 7/CentOS 7快速部署Kubernetes 1.7集群，http://dockone.io/article/2514
3. CentOS7.3利用kubeadm安装kubernetes1.7.3完整版(官方文档填坑篇)，https://www.cnblogs.com/liangDream/p/7358847.html
4. How to execute “kubeadm init” v1.6.4 behind firewall,https://stackoverflow.com/questions/44432328/how-to-execute-kubeadm-init-v1-6-4-behind-firewall
5. 使用 kubeadm 创建 kubernetes 1.9 集群，https://www.kubernetes.org.cn/3357.html

# 如有疑问，欢迎交流

