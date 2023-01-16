---
title: "Kubernetes教程(十一)---使用 KubeClipper 通过一条命令快速创建 k8s 集群"
description: "使用 KubeClipper 通过一条命令快速创建 k8s 集群"
date: 2022-08-20
draft: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
---

本文主要记录了如何使用开源项目 KubeClipper 通过一条命令快速创建 k8s 集群。

<!--more-->

>  2022-12-12，[KubeClipper](https://github.com/kubeclipper) 发布[ release-1.3.1](https://github.com/kubeclipper/kubeclipper/releases/tag/v1.3.1) 版本，本文也同步进行更新
>
>  **1.3.1 版本最大提升在于部署体验以及安装速度上的优化，同时一些错误提示也更加人性化，使用体验方便有较大提升**。
>
>  **部署参数大幅减少**：1.3.1 版本 aio 部署甚至不需要任何参数，kcctl deploy 即可完成部署，多节点部署也只需要提供两三个参数即可，较之前体验上有较大提升。
>
>  **集群安装速度大幅提升**：之前单节点集群需要 3 分钟，5节点高可用集群需要 9分钟，1.3.1 版本单节点只需要两分钟，5 节点也只需要4分钟。整个部署时间并不会随着节点增加而线性增加。

之前写了一篇使用 kubeadm 创建集群的文章： [Kubernetes教程(一)---使用 kubeadm 创建 k8s 集群(containerd)](https://www.lixueduan.com/posts/kubernetes/01-install/)，整个流程下来其实还是比较麻烦的，体验并不是太好，今天给大家推荐一个优秀的开源项目：[KubeClipper ](https://github.com/kubeclipper/kubeclipper)。



## 1. 什么是 KubeClipper

KubeClipper 旨在提供易使用、易运维、极轻量、生产级的 Kubernetes 多集群全生命周期管理服务，让运维工程师从繁复的配置和晦涩的命令行中解放出来，实现一站式管理跨区域、跨基础设施的多 K8S 集群。

KubeClipper 的 3个优势：

* 1）易使用：图形化页面
  * 提供 Web 控制台，通过点点鼠标即可创建一个 k8s 集群
* 2）极轻量：架构简单，少依赖
  * 不依赖 ansible，使用方便
* 3）生产级：易用性和专业性兼顾
  * 提供在线、离线、代理方式部署以及多版本 K8S、CRI、CNI 选择
  * 提供了离线部署方式对**国内用户极为友好**，毕竟网络才是国内用户装 k8s 的一大难题。

关于该项目的更多信息见：[Github Repo](https://github.com/kubeclipper/kubeclipper) 以及官方的这篇介绍文章：[KubeClipper——轻量便捷的 Kubernetes 多集群全生命周期管理工具](https://mp.weixin.qq.com/s/RVUB5Pw6-A5zZAQktl8AcQ)

一句话概括：**KubeClipper 是一个轻量便捷的 Kubernetes 多集群全生命周期管理工具**。



## 2. 快速开始

### 准备工作

KubeClipper 本身并不会占用太多资源，但是为了后续更好的运行 Kubernetes 建议硬件配置不低于最低要求。

您仅需参考以下对机器硬件和操作系统的要求准备一台主机。

#### 硬件推荐配置

- 确保您的机器满足最低硬件要求：CPU >= 2 核，内存 >= 2GB。
- 操作系统：CentOS 7.x / Ubuntu 18.04 / Ubuntu 20.04。

#### 节点要求

- 节点必须能够通过 `SSH` 连接。
- 节点上可以使用 `sudo` / `curl` / `wget` / `tar` / `gzip`命令。

> 建议您的操作系统处于干净状态（不安装任何其他软件），否则可能会发生冲突。



实际上这些要求基本上不用特别处理，随便找一台机器都满足条件，本文就使用一台 2C4G 的 CentOS 7.9 机器来部署。



### 安装 KubeClipper

#### 下载 kcctl

KubeClipper 提供了命令行工具🔧 kcctl，我们先安装 kcctl，安装命令如下：

```bash
curl -sfL https://oss.kubeclipper.io/kcctl.sh | KC_REGION=cn VERSION=v1.3.1 bash -
```

通过以下命令检测是否安装成功:

```bash
kcctl version
```



#### 开始安装

然后使用 kcctl 安装 kubeclipper，一条命令即可,模板如下：

```bash
kcctl deploy
```

kcctl 将检查安装环境，若满足条件则会在当前节点安装 kubeclipper，在打印出如下的 KubeClipper banner 后即表示安装完成。

```
 _   __      _          _____ _ _
| | / /     | |        /  __ \ (_)
| |/ / _   _| |__   ___| /  \/ |_ _ __  _ __   ___ _ __
|    \| | | | '_ \ / _ \ |   | | | '_ \| '_ \ / _ \ '__|
| |\  \ |_| | |_) |  __/ \__/\ | | |_) | |_) |  __/ |
\_| \_/\__,_|_.__/ \___|\____/_|_| .__/| .__/ \___|_|
                                 | |   | |
                                 |_|   |_|
```

> 安装过程中需要去阿里云下载离线安装包，大概 1 分钟即可下载完成。



### 登录控制台

安装完成后，打开浏览器，访问 `http://$IP` 即可进入 KubeClipper 控制台。

![](../../../img/kubernetes/kubeclipper/kc-console-login.png)

您可以使用默认帐号密码 `admin / Thinkbig1` 进行登录。

> 您可能需要配置端口转发规则并在安全组中开放端口，以便外部用户访问控制台。

> 文中所有操作均通过命令行工具完成，至于控制台相关功能则由大家自行探索了~



###  安装 K8S 集群

部署成功后可以使用 **kcctl 工具**或者通过**控制台**创建 k8s 集群， 这里咱们使用 kcctl 工具进行创建。

> 命令行，程序猿最后的倔强。

首先使用默认账号密码进行登录获取 token，便于后续 kcctl 和 kc-server 进行交互。

```bash
kcctl login -H http://localhost -u admin -p Thinkbig1
```

查看当前 agent 节点

```bash
[root@iZbp1cz9txcv4n6uluew1jZ ~]# kcctl get node
+--------------------------------------+----------+---------+----------------+-------------+-----+--------+
|                  ID                  | HOSTNAME | REGION  |       IP       |   OS/ARCH   | CPU |  MEM   |
+--------------------------------------+----------+---------+----------------+-------------+-----+--------+
| f22d488f-af17-47cb-ab55-07f8c4cce5f0 | server1  | default | 172.20.175.140 | linux/amd64 |   2 | 3645Mi |
+--------------------------------------+----------+---------+----------------+-------------+-----+--------+

```

然后使用以下命令创建 k8s 集群:

```bash
# 其中 IP 为上一步中的 server1 节点 IP
kcctl create cluster --name demo --master 172.20.175.140 --untaint-master
```

大概两分钟左右即可完成集群创建,可以使用以下命令查看实时日志：

```bash
while true;
do 
logDir="/var/log/kc-agent";
op=$(ll $logDir -t|grep -v total|head -n 1|awk '{print $9}');
echo -e "\n当前操作 ID："$op;
latestSteps=$(ll "$logDir/$op" -t|grep -v total|head -n 1|awk '{print $9}');
echo -e "当前最新步骤："$latestSteps"\n";
tail -n 20 "$logDir/$op/$latestSteps";
sleep 3;
done
```

或者使用以下命令查看集群状态

```bash
kcctl get cluster -o yaml|grep status -A5
```

> 也可以进入控制台查看**实时日志**。

进入 Running 状态即表示集群安装完成,您可以使用 `kubectl get cs` 命令来查看集群健康状况。

```bash
$ kubectl get cs
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""} 
```



至此，我们的单节点版本就算是结束了，相比之下确实比使用原生 kubeadm 体验好多了。



## 3. 多节点版本

为了保证高可用，k8s 集群至少需要 3个 master 节点，同时 kubeclipper 使用 etcd 作为后端存储，为了保证高可用，也建议至少使用 3 节点及以上来部署。

>  同时生产环境一般建议 kubeclipper server 节点单独存在，不建议将 server 节点同时作为 agent 节点使用。

节点规划如下：

* 3 台 作为 kubeclipper server 部署使用
* 5 台 作为 kubeclipper agent 节点，部署 k8s 使用。

> 资源有限，依旧是 2C4G 的 CentOS 7.9 机器



### 安装 KubeClipper

同样的，还是需要先安装 kcctl 工具，安装方式和之前是一样的，一条命令搞定：

> kcctl 工具只需要在任意一台节点上安装即可，只要能使用 ssh 访问其他节点即可。
>
> 为了和本教程保持一致，建议在其中的一台 server 节点上安装。

```bash
curl -sfL https://oss.kubeclipper.io/kcctl.sh | KC_REGION=cn VERSION=v1.3.1 bash -
```



接着使用 kcctl 部署 KubeClipper，其模板如下所示：

```bash
kcctl deploy  [--user root] (--passwd SSH_PASSWD | --pk-file SSH_PRIVATE_KEY) (--server SERVER_NODES) (--agent AGENT_NODES)
```

> 需要手动指定 server 节点以及 agent 节点。

若使用 密码 方式则命令如下所示:

```bash
kcctl deploy --passwd $SSH_PASSWD --server SERVER_NODES --agent AGENT_NODES
```

私钥 方式如下：

```bash
kcctl deploy --pk-file $SSH_PRIVATE_KEY --server SERVER_NODES --agent AGENT_NODES
```

本教程使用 密码 方式进行部署，具体命令如下：

```bash
# 3 个 server 节点，5 个 agent 节点，多个 IP 之间使用逗号分隔
servers=172.20.175.140,172.20.175.142,172.20.175.146
agents=172.20.175.145,172.20.175.143,172.20.175.144,172.20.175.139,172.20.175.141

kcctl deploy --server $servers --agent $agents --passwd Thinkbig1
```



同样的，在打印出如下的 KubeClipper banner 后即表示安装完成。

```text
 _   __      _          _____ _ _
| | / /     | |        /  __ \ (_)
| |/ / _   _| |__   ___| /  \/ |_ _ __  _ __   ___ _ __
|    \| | | | '_ \ / _ \ |   | | | '_ \| '_ \ / _ \ '__|
| |\  \ |_| | |_) |  __/ \__/\ | | |_) | |_) |  __/ |
\_| \_/\__,_|_.__/ \___|\____/_|_| .__/| .__/ \___|_|
                                 | |   | |
                                 |_|   |_|
```



### 创建高可用 K8S 集群

这里我们同样使用 kcctl 工具，通过命令行方式进行创建。老规矩，使用默认账号密码进行登录：

```bash
kcctl login -H http://localhost -u admin -p Thinkbig1
```

然后查看一下当前的 agent 节点信息：

```bash
[root@server1 ~]# kcctl get node
+--------------------------------------+----------+---------+----------------+-------------+-----+--------+
|                  ID                  | HOSTNAME | REGION  |       IP       |   OS/ARCH   | CPU |  MEM   |
+--------------------------------------+----------+---------+----------------+-------------+-----+--------+
| a0e33d98-fd43-4e69-9700-3bc201b16a3b | agent1   | default | 172.20.175.145 | linux/amd64 |   2 | 3645Mi |
| 4a62dc30-b1a5-42b3-87e2-373733e6ae71 | agent2   | default | 172.20.175.143 | linux/amd64 |   2 | 3645Mi |
| b5a45667-38fd-4448-ad59-2c93bc049f37 | agent5   | default | 172.20.175.141 | linux/amd64 |   2 | 3645Mi |
| 2f5a889a-8ee3-4f7d-9a41-ba55761c89bc | agent4   | default | 172.20.175.139 | linux/amd64 |   2 | 3645Mi |
| ea9790f7-add6-4f96-997a-6ef1061b6907 | agent3   | default | 172.20.175.144 | linux/amd64 |   2 | 3645Mi |
+--------------------------------------+----------+---------+----------------+-------------+-----+--------+
```

接下来就可以使用 `kcctl create cluster` 命令创建集群了,命令模版如下：

```bash
kcctl create cluster (--name CLUSTER_NAME) (--master NODES) [--worker NODES][ --untaint-master] 
```

> 更多信息见 `kcctl create cluster -h`

在本教程中，我们将 agent1、2、3 作为 master 节点，agent4、5作为 worker 节点，完整命令如下：

```bash
masters=172.20.175.145,172.20.175.143,172.20.175.144
workers=172.20.175.139,172.20.175.141

kcctl create cluster --name demo --master $masters --worker $workers
```

对于这样一个 5 节点的集群大概 3 到 4 分钟即可创建完成。

可以 ssh 到第一个 master 节点对应的 agent 节点（即 agent1）使用以下命令查看实时日志：

```bash
while true;
do 
logDir="/var/log/kc-agent";
op=$(ll $logDir -t|grep -v total|head -n 1|awk '{print $9}');
echo -e "\n当前操作 ID："$op;
latestSteps=$(ll "$logDir/$op" -t|grep -v total|head -n 1|awk '{print $9}');
echo -e "当前最新步骤："$latestSteps"\n";
tail -n 20 "$logDir/$op/$latestSteps";
sleep 3;
done
```

或者使用以下命令查看集群状态

```bash
kcctl get cluster -o yaml|grep status -A5
```

> 也可以进入控制台查看**实时日志**。



进入 Running 状态即表示集群安装完成,您可以使用 `kubectl get cs` 命令来查看集群健康状况。

**ssh 到任意 master 节点**上查看集群状态：

```bash
[root@agent1 ~]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""}   
[root@agent1 ~]# kubectl get node
NAME     STATUS   ROLES                  AGE     VERSION
agent1   Ready    control-plane,master   3m38s   v1.23.6
agent2   Ready    control-plane,master   3m13s   v1.23.6
agent3   Ready    control-plane,master   3m12s   v1.23.6
agent4   Ready    <none>                 2m18s   v1.23.6
agent5   Ready    <none>                 2m17s   v1.23.6
```

可以看到，5 节点都处于 Ready 状态，说明集群安装成功。



## 4. 小结

本文演示了如何通过 KubeClipper 使用简单的命令创建 K8S 集群。

整个流程可以简单的分为 3 个步骤：

* 1）安装 kcctl

```bash
#不指定VERSION则默认安装master分支构建版本
curl -sfL https://oss.kubeclipper.io/kcctl.sh | KC_REGION=cn VERSION=v1.3.1 bash -
```

* 2）使用 kcctl 部署 kubeclipper

```bash
kcctl deploy [--user root] (--passwd SSH_PASSWD | --pk-file SSH_PRIVATE_KEY) (--server SERVER_NODES) (--agent AGENT_NODES)
```

* 3）使用 kcctl 或控制台创建 K8S 集群

```bash
kcctl create cluster (--name CLUSTER_NAME) (--master NODES) [--worker NODES][ --untaint-master] 
```

文中所有操作均通过命令行工具完成，至于控制台相关功能则由大家自行探索了~



> 如果觉得体验还不错的话，请不要吝啬你的 star ，[KubeClipper](https://github.com/kubeclipper) 还处于快速成长阶段，欢迎大家积极参与~