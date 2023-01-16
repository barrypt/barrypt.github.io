---
title: "Kubernetes教程(十五)---使用 kind 在本地快速部署一个 k8s集群"
description: "通过Kubeadm部署K8s集群"
date: 2022-12-10
draft: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
---

本文主要记录了如何使用 kind 在本地快速部署一个 k8s集群，包括 kind 的基本使用、大致原理以及注事事项等。

<!--more-->

> repo：[https://github.com/kubernetes-sigs/kind](https://github.com/kubernetes-sigs/kind)
>
> website：[https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/)



**一句话描述，什么是 kind**：

**[kind](https://sigs.k8s.io/kind)** **is a tool for running local Kubernetes clusters using Docker container “nodes”.**

Kind 是一个通过使用 docker 容器模拟节点来创建本地 k8s 集群的工具。



## 1. QuickStart

### Install docker

Kind 使用 docker 来启动容器，因此需要先安装 docker，可自行安装，也可以用下面这个脚本安装。

通过以下命令创建 install-docker.sh  脚本文件：

```Bash
cat << EOF > install-docker.sh 
#!/bin/bash
set -e
# centos 一键安装 docker 脚本.

#卸载旧版本
yum remove -y docker  docker-common docker-selinux docker-engine
#安装需要的软件包
yum install -y yum-utils device-mapper-persistent-data lvm2
#添加yum源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#安装最新版docker
yum install -y docker-ce
#配置镜像加速器
mkdir -p /etc/docker
echo '{
  "registry-mirrors": [
    "https://reg-mirror.qiniu.com",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}' > /etc/docker/daemon.json
#启动并加入开机启动
systemctl enable docker --now
docker version
echo "docker install finish."
EOF
```

运行脚本，安装 docker：

```Bash
sh install-docker.sh
```



### Install kind from binary

kind 只是一个二进制文件，因此下载下来放到 bin 目录即可。

```Bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```



### Install kubectl

Kind 只负责创建集群（会配置好 kubeconfig），后续操作集群的话需要手动安装 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

```Bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin
```



### Create aio cluster

一切就绪之后，使用 kind 创建集群即可。

> kind 0.17.0 默认用的是 v.1.25.3 版本 k8s，可以先手动拉镜像,因为默认的镜像地址不一定能拉下来。

```Bash
docker pull kindest/node:v1.25.3
```

然后创建集群时指定刚才拉下来的镜像

```Bash
kind create cluster --image kindest/node:v1.25.3 --name aio -v 5
```

不出意外的话，一两分钟就可以创建好。

创建好之后就可以使用 kubectl 进行操作了，kind 会把 kubeconfig 也配置好，使用起来和真正的集群基本一致。



### List cluster

Kind 自身也保存了集群数据的，使用以下命令进行查看。

```Bash
kind get clusters
```



### Delete cluster

删除一个集群也很简单，通过集群名即可

```Bash
# syntax：kind delete  cluster --name ${clustername}
kind delete  cluster --name aio
```



### Create ha cluster

默认创建的集群只有一个 master 节点，不过可以通过配置文件创建 HA 集群。

使用以下命令生成一个 kind-ha.yaml  配置文件：

> 配置文件内容很简单，nodes 字段里多指定几个节点就行了。

```YAML
cat <<EOF > kind-ha.yaml
# a cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
```

以上配置文件指定了创建一个 3 master 3 worker 的 k8s 集群。同时，在 HA master 下， 它还额外部署了一个 Nginx，用来提供负载均衡 vip。

> **注意：这里的 HA 并不是真正意义上的 HA，毕竟所有 node 都跑在一个节点上的，如果底层的硬件或者 Docker 出问题那么整个集群都会挂掉。**
>
> 因此仅供本地测试使用。



创建时通过 `--config` 指定该配置文件：

```Bash
kind create cluster --image kindest/node:v1.25.3 --name ha --config kind-ha.yaml
```



## 2. 实现原理

**Kind 使用一个docker 容器来模拟一个 node，在 docker 容器里面跑 systemd ，并用 systemd 托管 kubelet 以及 containerd，然后通过容器内部的 kubelet 把其他 K8s 组件，比如 kube-apiserver、etcd 等跑起来，最后在部署上 CNI 整个集群就完成了**

> Kind 内部也是使用的 kubeadm 进行部署。



我们可以进入到 docker 容器进行查看：

```Bash
[root@kind ~]# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                       NAMES
c9a55b2ea333   kindest/node:v1.25.3   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:33275->6443/tcp   aio-control-plane
[root@kind ~]# docker exec -it c9a55b2ea333 /bin/bash
```

先看下 containerd

```bash
root@aio-control-plane:/# systemctl status containerd
● containerd.service - containerd container runtime
     Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-12-10 04:42:01 UTC; 4min 8s ago
       Docs: https://containerd.io
   Main PID: 243 (containerd)
      Tasks: 122
     Memory: 799.4M
     CGroup: /docker/c9a55b2ea333c9fd7ffcce78a6a96c0b1c9d618f4ecdf1bf3568334684b7186c/system.slice/containerd.service
             ├─ 243 /usr/local/bin/containerd
             ├─ 371 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 648e35fa086d1e73a07688fa742be18ee3cbd72f12412153329fd0cf43693b7e -address /run/containerd/containerd.sock
             ├─ 398 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id c9b803db61f0194f84e451615dea5cd463b921590b83d044ef7ecdce7c62608e -address /run/containerd/containerd.sock
             ├─ 401 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 831789d91a8142289e4263eb0dbba4b9ff4b9d36f930c41248c4fc77e3aeeb6d -address /run/containerd/containerd.sock
             ├─ 429 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 25d98fdf10c4198c8ddcf7065eed292bcc6a0b1dfd069b640a50b5f307a551de -address /run/containerd/containerd.sock
             ├─ 862 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id b880b9a217fb747602beadc45890efbee81fadc8d6d6d5487c0428c3c5e2a1fc -address /run/containerd/containerd.sock
             ├─ 888 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 79a95b6fa3be26c317fe8e4144420caa0db537ed5fb748be28acf497fdc15a8e -address /run/containerd/containerd.sock
             ├─1208 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 0eb055283fcef14662c9e4e4497f33ea9b506f2455df2c8a7839a0fce6b83f1d -address /run/containerd/containerd.sock
             ├─1231 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 0e58541ef79169ce4f17f27493e671e90c940c52e12b786faaf108aaa742c8a9 -address /run/containerd/containerd.sock
             └─1311 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 6689352dad1bfa8e57522f231deefd8fd2169a765f428b6ad1c1c58b82c8dbf1 -address /run/containerd/containerd.sock
```

再看下 kubelt

```bash
root@aio-control-plane:/# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Sat 2022-12-10 04:42:32 UTC; 55s ago
       Docs: http://kubernetes.io/docs/
    Process: 748 ExecStartPre=/bin/sh -euc if [ -f /sys/fs/cgroup/cgroup.controllers ]; then create-kubelet-cgroup-v2; fi (code=exited, status=0/SUCCESS)
    Process: 749 ExecStartPre=/bin/sh -euc if [ ! -f /sys/fs/cgroup/cgroup.controllers ] && [ ! -d /sys/fs/cgroup/systemd/kubelet ]; then mkdir -p /sys/fs/cgroup/systemd/kubelet; fi (code=exited, status=0/SUCCESS)
   Main PID: 750 (kubelet)
      Tasks: 14 (limit: 4662)
     Memory: 40.1M
        CPU: 2.485s
     CGroup: /docker/c9a55b2ea333c9fd7ffcce78a6a96c0b1c9d618f4ecdf1bf3568334684b7186c/kubelet.slice/kubelet.service
             └─750 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --node-ip=172.18.0.2 --node-labels= --pod-infra-container-image=registry.k8s.io/pause:3.8 --provider-id=kind://docker/aio/aio-control-plane --fail-swap-on=false --cgroup-root=/kubelet
```

可以看到，在容器里确实以 systemd 方式启动了一个containerd 和 kubelet，然后其他组件以静态 pod 的方式被 kubelet 拉起。

然后 cni 的话 kind 使用的是自己实现的一个简单的 cni，叫做  [kindnet](https://github.com/kubernetes-sigs/kind/tree/main/images/kindnetd)

```Bash
root@aio-control-plane:/# kubectl -n kube-system get ds
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kindnet      1         1         1       1            1           <none>                   7m32s
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   7m34s
```





## 3. 注意事项

由于 kind 是通过 docker 容器模拟 node 来部署集群的，因此和普通集群有一些差异。主要包括以下几个方面：

- 文件系统
  - kind cluster 中无法直接访问宿主机上的文件
  - kind cluster 中无法直接使用宿主机上的镜像

- 网络
  - 比如无法在宿主机直接访问 kind cluster 中的服务

以上问题 kind 都提供了相应的解决方案，比如镜像导入、端口映射、目录映射等等，具体见下文。



### 镜像导入

Q：宿主机上使用 docker images 查看明明有镜像，但是 kubectl apply 却要去拉镜像？

A：这是由于 kind 中的节点实际都是 docker 容器导致的，和宿主机是隔离的。**在宿主机拉取镜像后还需要通过 kind load  方式加载到 node 里面（容器里）去**,否则部署的应用会再次拉取镜像。

相关命令如下：

> 以下命令都是在宿主机上执行

```Bash
# 查看 node 里的镜像列表
docker exec -it dev-control-plane crictl images

# 导入镜像到指定集群
kind load docker-image caas4/etcd:3.5.5 --name aio
```

### 

### 如何访问集群中的服务

由于 kind 是把 node 运行在 docker 里的， 因此即使在 k8s 集群中使用 nodePort 方式将服务暴露出来，在宿主机上依旧无法访问。

> 因为这里暴露的 nodePort 其实是 docker 容器的端口，如果 docker 容器启动时没有将端口映射出来依旧无法访问

有两种解决方案：

- 进入到对应 network namespace

- 端口映射



#### 进入到对应 network namespace

由于kind 集群的 node 是以 docker 容器运行的，那么只需要进入到该容器的 network namespace 就可以访问到 集群中暴露出来的 nodePort。

> 当然，clusterIP 应该还是无法访问的。

相关命令如下：

```Bash
containerID=xxx
pid=$(docker inspect -f {{.State.Pid}} $containerID)
echo $pid
nsenter -n -t $pid
```



#### 端口映射

Kind 在**启动集群时**可以指定将某些端口映射出来，因为是通过 docker 跑的，所以后续应该只是把参数传给 docker 了。

> 类似于指定了 docker run -p 参数。
>
> 注意：由于 docker 容器启动后不能修改端口映射，因此 kind 集群创建后也不能修改端口映射了。

配置文件如下：

```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
```

在参数 extraPortMappings 中定义要映射的端口即可，多个节点则需要挨个定义。

定义了端口映射后再启动集群，对应的 docker 容器就会直接把端口暴露出来了

```Bash
[root@kind ~]# docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                                                                           NAMES
54c8bb471113   kindest/node:v1.25.3                 "/usr/local/bin/entr…"   10 minutes ago   Up 10 minutes   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:30000->30000/tcp, 127.0.0.1:42259->6443/tcp   portmapping-control-plane
```

这样就可以在宿主机上访问到集群中的服务了。



#### 小结

由于端口映射只能在创建集群时指定，创建后无法修改，因此在测试时建议直接使用方式一，进入到对应 network namespace 进行访问。

等后续端口固定好之后再通过端口映射一次性搞定。



### 目录映射

和端口映射类似，kind cluster 里的 hostpath 指的是 docker 里的文件系统，而想要用宿主机上的文件还需要再映射一次。

> 最终还是会把参数传递给 docker，类似于 docker run -v 参数。

同样只能在创建集群时指定，配置文件如下：

```YAML
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /home/bill/work/foo
        containerPath: /foo
```

通过 extraMounts 参数指定需要映射的目录。

通过该配置文件启动的 docker 容器就会挂载相应目录到容器里。

集群启动后进入 docker 容器查看，目录映射已经生效：

```Bash
[root@kind ~]# docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                       NAMES
5e10f9cbc1bd   kindest/node:v1.25.3                 "/usr/local/bin/entr…"   7 minutes ago    Up 7 minutes    127.0.0.1:42521->6443/tcp   volumemapping-control-plane
[root@kind ~]# docker exec -it 5e10f9cbc1bd ls /foo
# 这个目录是宿主机上的，能在容器里看到，说明成功了
systemd-private-668bfa87d90144a1a43345805f73cd6a-chronyd.service-Tuqj3U
```





## 4.小结

本文主要介绍了 如果快速使用 kind 在本地创建一个 k8s 集群，及其原理和注意事项等。

* 如果需要在本地测试的话推荐使用 [kind ](https://github.com/kubernetes-sigs/kind)
* 如果需要真正的创建集群的话现在主流工具则是 [kubeadm](https://github.com/kubernetes/kubeadm)



> 如果觉得 kubeadm 也比较麻烦的话可以试一下 [kubeclipper](https://github.com/kubeclipper/kubeclipper)，基于 kubeadm 工具进行二次封装, [完全兼容原生 Kubernetes](https://landscape.cncf.io/card-mode?category=certified-kubernetes-installer&grouping=category&selected=kube-clipper), 以**图形化**方式提供在企业自有基础设施中**快速部署 K8S 集群和持续化全生命周期管理**（安装、卸载、升级、扩缩容、远程访问等）能力。



