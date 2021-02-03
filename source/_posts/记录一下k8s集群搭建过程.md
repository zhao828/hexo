---
title: 记录一下k8s集群搭建过程
tags:
  - k8s
  - vmware
  - docker
author: Thzs
categories:
  - k8s
date: 2021-02-02 16:59:00
---


# 记录一下k8s集群搭建过程


## 虚拟机配置

我是在vm station 16上部署，由于笔记本只有16g内存，所以配置如下
{% note danger %}
cpu 在 k8s官方要求中需要2核
{% endnote %}


| HostName | CPU  | RAM  | RAM |
| -------- | ---- | ---- | --- |
| master   | 2    | 3    |20   | 
| worker1  | 2    | 3    |20   |
| worker2  | 2    | 3    |20   |

### 配置虚拟机网络

首先虚拟机网络默认是 `NAT模式`。为了测试方便，我们需要在虚拟机设置中改成桥接模式，并且勾选同步虚拟机与宿主机的时间。
{% note danger %}
切记，如果没有勾选时区同步。就要使用yum安装时间同步工具，让worker节点与master节点时间同步
{% endnote %}

首先 `ip address` 查看自己网卡名字，然后运行 

```   
    vi /etc/sysconfig/network-scripts/ifcfg-网卡名字
```
然后 在 `ONBOOT=no` 修改成yes就行了。再运行下`ip address`可以看到自己的ip地址了

对每一台都配置master和worker节点的域名
```  
    vi /etc/hosts
     ip1 master
     ip2 node1
     ip3 node2
```

### 配置虚拟机环境
关闭 防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```

关闭 SeLinux
```
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

关闭 swap
```
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab
```


## 开始搭建
### 软件版本

**centos 7-minimal**

**docker 18.09.4**

**k8s 1.20.2**

---
### 配置docker

安装依赖
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```
添加docker repo
``` 
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
{% note secondary %}
国内可以配置阿里云
``` 
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
{% endnote %}
开始安装
```
yum install -y docker-ce-18.09.4 docker-ce-cli-18.09.4
```
docker 默认是cgroup，需要更改让`systemd`作为 cgroup驱动使得更加稳定
```
vi usr/lib/systemd/system/docker.service
```
找到cgroupdirver，改为下面的systemd
`cgroupdriver=systemd`

然后运行`docker info`查看配置是否成功

设置docker开机启动并启动
```
systemctl enable docker
systemctl start docker
```
---
### 配置k8s
这里使用原生包安装器安装，国内可以把里面`baseurl`换成阿里即可
#### master节点
安装k8s三兄弟
``` 
yum install -y kubelet kubeadm kubectl
```
安装完成后设置开机启动并启动
```
systemctl enable kubelet && systemctl start kubelet 
```

接下来是对于 **master节点** 的初始化

```
 kubeadm init --kubernetes-version=1.20.2 \
 --apiserver-advertise-address=自己的ip \
 --image-repository k8s.gcr.io \ 可以改成阿里的 registry.aliyuncs.com/google_containers
 --service-cidr=10.1.0.0/16 \
 --pod-network-cidr=10.244.0.0/16
```
如果你是普通用户，非root那么要运行
``` 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
安装完成后会给你一个kubectl join的命令加token，找个记事本记下来。等会work节点加入集群要用。

安装flannel网络插件
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
#### worker节点
和上面一样，除了不用kubectl init。如果你需要在worker节点里运行kubectl 命令，还需要把master节点配置拷过来，这里不做阐述可以自行谷歌。


---

## 测试

在上面所有安装完成后运行```kubectl get nodes``` 查看是否ready，不ready可能是网络查件问题。

并且运行 `kubectl get cs` 查看controller是否健康，如果`scheduler` 和 `controller-manger` 不健康，
那么就要修改`/etc/kubernetes/manifests/kube-controller-manager.yaml和kube-scheduler.yaml`把`port = 0`注释掉。

### 部署一个nginx 测试下

```
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```
检查pod是否运行正常
```
kubectl get pods -l app=nginx
```
检查deployment部署是否正常
``` 
kubectl get deploy
```
如果都是ready 那么恭喜部署成功啦
如果有问题 多用 `kubectl describe deployment [deployname] 和 kubectl logs [podname]` 检查下什么问题