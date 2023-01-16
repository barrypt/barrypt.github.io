---
title: "Docker教程(九)---如何使用UFS(overlayfs)实现rootfs"
description: "Union File System中的 overlayfs 演示，以及docker 通过 overlayfs构建 rootfs流程分析"
date: 2022-03-19 22:00:00
draft: false
tags: ["Docker"]
categories: ["Docker"]
---

本文主要介绍了 Docker 的另一个核心技术：Union File System。主要包括对 overlayfs  的演示，以及分析 docker 是如何借助 ufs 实现容器 rootfs 的。

<!--more-->



> 跟着《自己动手写 docker》从零开始实现了一个简易版的 docker，主要用于加深对 docker 的理解。
>
> 源码及相关教程见 [Github][Github]。



[Github]:https://github.com/lixd/mydocker

## 1. 概述

### Union File System

Union File System ，简称 UnionFS 是一种为 Linux FreeBSD NetBSD 操作系统设计的，把其他文件系统联合到一个联合挂载点的文件系统服务。

它使用 branch 不同文件系统的文件和目录“透明地”覆盖，形成 个单一一致的文件系统。这些branches或者是read-only或者是read-write的，所以当对这个虚拟后的联合文件系统进行写操作的时候，系统是真正写到了一个新的文件中。看起来这个虚拟后的联合文件系统是可以对任何文件进行操作的，但是其实它并没有改变原来的文件，这是因为unionfs用到了一个重要的资管管理技术叫写时复制。

**写时复制（copy-on-write，下文简称CoW）**，也叫隐式共享，是一种对可修改资源实现高效复制的资源管理技术。它的思想是，如果一个资源是重复的，但没有任何修改，这时候并不需要立即创建一个新的资源；这个资源可以被新旧实例共享。创建新资源发生在第一次写操作，也就是对资源进行修改的时候。通过这种资源共享的方式，可以显著地减少未修改资源复制带来的消耗，但是也会在进行资源修改的时候增减小部分的开销。

**举个例子**

假设我们存在2个目录X,Y，里面分别有A，B文件，那么UFS的作用就是将这两个目录合并，并且重新挂载的Z上,这样在Z目录上就可以同时看到A和B文件。这就是联合文件系统，目的就是**将多个文件联合在一起成为一个统一的视图**。

然后我们在Z目录中删除B文件，同时，在A文件中增加一些内容，如添加Hello字符串。此时可以发现，X内的A文件新增了Hello,并且新增了一条B被删除的记录，但是Y中的B并没有任何变化。这是UFS的一个重要特性。**在所有的联合起来的目录中，只有第一个目录是有写的权限**，即我们不管如何的去对Z进行修改操作，都只能对第一个联合进来的X修改，对Y是没有权限修改的。

但是如果我们在Z中对Y中的文件进行了修改，它虽然没有权限去修改Y目录中的文件，但是它会在第一层目录添加一个记录来记录更改内容。



Union File System 也叫 UnionFS，最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下。比如，我现在有两个目录 A 和 B，它们分别有两个文件：

```shell
$ tree
.
├── A
│  ├── a
│  └── x
└── B
  ├── b
  └── x
```

然后，我使用联合挂载的方式，将这两个目录挂载到一个公共的目录 C 上：

```shell
$ mkdir C
$ mount -t aufs -o dirs=./A:./B none ./C
```

这时，我再查看目录 C 的内容，就能看到目录 A 和 B 下的文件被合并到了一起：

```shell
$ tree ./C
./C
├── a
├── b
└── x
```

可以看到，在这个合并后的目录 C 里，有 a、b、x 三个文件，并且 x 文件只有一份。这，就是“合并”的含义。此外，如果你在目录 C 里对 a、b、x 文件做修改，这些修改也会在对应的目录 A、B 中生效。



### 常见实现

#### AUFS

AuFS 的全称是 Another UnionFS，后改名为 Alternative UnionFS，再后来干脆改名叫作 Advance UnionFS。AUFS完全重写了早期的UnionFS 1.x，其主要目的是为了可靠性和性能，并且引入了一些新的功能，比如可写分支的负载均衡。AUFS的一些实现已经被纳入UnionFS 2.x版本。

AUFS 只是 Docker 使用的存储驱动的一种，除了 AUFS 之外，Docker 还支持了不同的存储驱动，包括 `aufs`、`devicemapper`、`overlay2`、`zfs` 和 `vfs` 等等，在最新的 Docker 中，`overlay2` 取代了 `aufs` 成为了推荐的存储驱动，但是在没有 `overlay2` 驱动的机器上仍然会使用 `aufs` 作为 Docker 的默认驱动。



#### overlayfs

Overlayfs 是一种类似 aufs 的一种堆叠文件系统，于 2014 年正式合入 Linux-3.18 主线内核，目前其功能已经基本稳定（虽然还存在一些特性尚未实现）且被逐渐推广，特别在容器技术中更是势头难挡。

Overlayfs 是一种堆叠文件系统，它依赖并建立在其它的文件系统之上（例如 ext4fs 和 xfs 等等），并不直接参与磁盘空间结构的划分，仅仅将原来底层文件系统中不同的目录进行“合并”，然后向用户呈现。



简单的总结为以下3点：

（1）上下层同名目录合并；

（2）上下层同名文件覆盖；

（3）lower dir文件写时拷贝。

这三点对用户都是不感知的。

假设我们有 dir1 和 dir2 两个目录：

```shell
  dir1                    dir2
    /                       /
      a                       a
      b                       c
```

然后我们可以把 dir1 和 dir2 挂载到 dir3上，就像这样：

```shell
 dir3
    /
      a
      b
      c
```

需要注意的是：在 overlay 中 dir1 和 dir2 是有上下关系的。lower 和 upper 目录不是完全一致，有一些区别，具体见下一节。



## 2. overlayfs 演示

### 环境准备

具体演示如下：

创建一个如下结构的目录：

```shell
.
├── lower
│   ├── a
│   └── c
├── merged
├── upper
│   ├── a
│   └── b
└── work
```

具体命令如下：

```shell
$ mkdir ./{merged,work,upper,lower}
$ touch ./upper/{a,b}
$ touch ./lower/{a,c}
```

然后进行 mount 操作：

```shell
# -t overlay 表示文件系统为 overlay
# -o lowerdir=./lower,upperdir=./upper,workdir=./work 指定 lowerdir、upperdir以及 workdir这3个目录。
# 其中 lowerdir 是自读的，upperdir是可读写的，
$ sudo mount \
            -t overlay \
            overlay \
            -o lowerdir=./lower,upperdir=./upper,workdir=./work \
            ./merged
```

此时目录结构如下：

```shell
.
├── lower
│   ├── a
│   └── c
├── merged
│   ├── a
│   ├── b
│   └── c
├── upper
│   ├── a
│   └── b
└── work
    └── work
```

可以看到，merged 目录已经可以同时看到 lower和upper中的文件了，而由于文件a同时存在于lower和upper中，因此被覆盖了，只显示了一个a。





### 修改文件

虽然 lower 和 upper 中的文件都出现在了 merged 目录，但是二者还是有区别的。

lower 为底层目录，只提供数据，不能写。

upper 为上层目录，是可读写的。



测试：

```shell
# 分别对 merged 中的文件b和c写入数据
# 其中文件 c 来自 lower，b来自 upper
$ echo "will-persist"  > ./merged/b
$ echo "wont-persist"  > ./merged/c
```

修改后从 merged 这个视图进行查看：

```shell
$ cat ./merged/b
will-persist
$ cat ./merged/c
wont-persist
```

可以发现，好像两个文件都被更新了，难道上面的结论是错的？

再从 upper 和 lower 视角进行查看：

```shell
$ cat ./upper/b
will-persist
$ cat ./lower/c
(empty)
```

可以发现 lower 中的文件 c 确实没有被改变。

那么 merged 中查看的时候，文件 c 为什么有数据呢？



由于 lower 是不可写的，因此采用了 CoW 技术，在对 c 进行修改时，复制了一份数据到 overlay 的 upperdir，即这里的 upper 目录，进入 upper 目录查看是否存在 c 文件：

```shell
[root@iZ2zefmrr626i66omb40ryZ upper]$ ll
total 8
-rw-r--r-- 1 root root  0 Jan 18 18:50 a
-rw-r--r-- 1 root root 13 Jan 18 19:10 b
-rw-r--r-- 1 root root 13 Jan 18 19:10 c
[root@iZ2zefmrr626i66omb40ryZ upper]$ cat c 
wont-persist
```



> 从 lower copy 到 upper，也叫做 copy_up



### 删除文件

首先往 lower 中增加文件 f

```shell
[root@iZ2zefmrr626i66omb40ryZ lower]$ echo fff >> f
[root@iZ2zefmrr626i66omb40ryZ lower]$ ls ../merged/
f
```

果然 lower 中添加后，merged 中也能直接看到了，然后再 merged 中去删除文件 f：

```shell
[root@iZ2zefmrr626i66omb40ryZ lower]$ cd ../merged/
[root@iZ2zefmrr626i66omb40ryZ merged]$ rm -rf f
# merged 中删除后 lower 中文件还在
[root@iZ2zefmrr626i66omb40ryZ merged]$ ls ../lower/
a  c  e  f
# 而 upper 中出现了一个大小为0的c类型文件f
[root@iZ2zefmrr626i66omb40ryZ merged]# ls -l ../upper/
total 0
c--------- 1 root root 0, 0 Jan 18 19:28 f
```

可以发现，overlay 中删除 lower 中的文件，其实也是在 upper 中创建一个标记，表示这个文件已经被删除了，测试一下：

```shell
[root@iZ2zefmrr626i66omb40ryZ merged]$ rm -rf ../upper/f
[root@iZ2zefmrr626i66omb40ryZ merged]$ ls
f
[root@iZ2zefmrr626i66omb40ryZ merged]$ cat f 
fff
```

把 upper 中的大小为0的f文件给删掉后，merged中又可以看到lower中 f 了，而且内容也是一样的。

说明 overlay 中的删除其实是**标记删除**。再 upper 中添加一个删除标记，这样该文件就被隐藏了，从merged中看到的效果就是文件被删除了。

> 删除文件或文件夹时，会在 upper 中添加一个同名的 `c` 标识的文件，这个文件叫 `whiteout` 文件。当扫描到此文件时，会忽略此文件名。



### 添加文件

最后再试一下添加文件

```shell
# 首先在 merged 中创建文件 g
[root@iZ2zefmrr626i66omb40ryZ merged]$ echo ggg >> g
[root@iZ2zefmrr626i66omb40ryZ merged]$ ls
g
# 然后查看 upper，发现也存在文件 g
[root@iZ2zefmrr626i66omb40ryZ merged]$ ls ../upper/
g
# 在查看内容，发送是一样的
[root@iZ2zefmrr626i66omb40ryZ merged]$ cat ../upper/g
ggg
```

说明 overlay 中添加文件其实就是在 upper 中添加文件。

测试一下删除会怎么样呢：

```shell
[root@iZ2zefmrr626i66omb40ryZ merged]$ rm -rf ../upper/g
[root@iZ2zefmrr626i66omb40ryZ merged]$ ls
f
```

把 upper 中的文件 g 删除了，果然 merged 中的文件 g 也消失了。





## 3. docker 是如何使用 overlay 的？

### 大致流程

每一个Docker image都是由一系列的read-only layers组成：

* image layers的内容都存储在Docker hosts filesystem的/var/lib/docker/aufs/diff目录下
* 而/var/lib/docker/aufs/layers目录则存储着image layer如何堆栈这些layer的metadata。



docker支持多种graphDriver，包括vfs、devicemapper、overlay、overlay2、aufs等等，其中最常用的就是aufs了，但随着linux内核3.18把overlay纳入其中，overlay的地位变得更重。

`docker info`命令可以查看docker的文件系统。

```sh
$ docker info 
# ...
 Storage Driver: overlay2
#...
```

比如这里用的就是 overlay2.



例如，假设我们有一个由两层组成的容器镜像：

```shell
   layer1:                 layer2:
    /etc                    /bin
      myconf.ini              my-binary
```

然后，在容器运行时将把这两层作为 lower 目录，创建一个空`upper`目录，并将其挂载到某个地方：

```shell
sudo mount \
            -t overlay \
            overlay \
            -o lowerdir=/layer1:/layer2,upperdir=/upper,workdir=/work \
            /merged
```

最后将`/merged`用作容器的 rootfs。

这样，容器中的文件系统就完成了。



### 具体分析

**以构建镜像方式演示以下 docker 是如何使用 overlayfs 的。**

先拉一下 Ubuntu:20.04 的镜像：

```shell
$ docker pull ubuntu:20.04
20.04: Pulling from library/ubuntu
Digest: sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322
Status: Downloaded newer image for ubuntu:20.04
docker.io/library/ubuntu:20.04
```



然后写个简单的 Dockerfile ：

```dockerfile
 FROM ubuntu:20.04

 RUN echo "Hello world" > /tmp/newfile
```

开始构建：

```shell
$ docker build -t hello-ubuntu .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM ubuntu:20.04
 ---> ba6acccedd29
Step 2/2 : RUN echo "Hello world" > /tmp/newfile
 ---> Running in ee79bb9802d0
Removing intermediate container ee79bb9802d0
 ---> 290d8cc1f75a
Successfully built 290d8cc1f75a
Successfully tagged hello-ubuntu:latest
```

查看构建好的镜像：

```shell
$ docker images
REPOSITORY                                             TAG            IMAGE ID       CREATED          SIZE
hello-ubuntu                                           latest         290d8cc1f75a   13 minutes ago   72.8MB
ubuntu                                                 20.04          ba6acccedd29   3 months ago     72.8MB
```

使用`docker history`命令，查看镜像使用的 image layer 情况：

```shell
$ docker history hello-ubuntu
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
290d8cc1f75a   22 seconds ago   /bin/sh -c echo "Hello world" > /tmp/newfile    12B       
ba6acccedd29   3 months ago     /bin/sh -c #(nop)  CMD ["bash"]                 0B        
<missing>      3 months ago     /bin/sh -c #(nop) ADD file:5d68d27cc15a80653…   72.8MB
```

> missing ”标记的 layer ，是自 Docker 1.10 之后，一个镜像的 image layer image history 数据都存储在 个文件中导致的，这是 Docker 官方认为 正常行为。

可以看到，290d8cc1f75a 这一层在最上面，只用了 12Bytes，而下面的两层都是共享的，这也证明了AUFS是如何高效使用磁盘空间的。

然后去找一下具体的文件：

docker默认的存储目录是`/var/lib/docker`,具体如下：

```shell
[root@iZ2zefmrr626i66omb40ryZ docker]$ ls -al
total 24
drwx--x--x  13 root root   167 Jul 16  2021 .
drwxr-xr-x. 42 root root  4096 Oct 13 15:07 ..
drwx--x--x   4 root root   120 May 24  2021 buildkit
drwx-----x   7 root root  4096 Jan 17 20:25 containers
drwx------   3 root root    22 May 24  2021 image
drwxr-x---   3 root root    19 May 24  2021 network
drwx-----x  53 root root 12288 Jan 17 20:25 overlay2
drwx------   4 root root    32 May 24  2021 plugins
drwx------   2 root root     6 Jul 16  2021 runtimes
drwx------   2 root root     6 May 24  2021 swarm
drwx------   2 root root     6 Jan 17 20:25 tmp
drwx------   2 root root     6 May 24  2021 trust
drwx-----x   5 root root   266 Dec 29 14:31 volumes
```

在这里，我们只关心`image`和`overlay2`就足够了。

* image：镜像相关
* overlay2：docker 文件所在目录，也可能不叫这个名字，具体和文件系统有关，比如可能是 aufs 等。

先看 `image`目录：

docker会在`/var/lib/docker/image`目录下按每个存储驱动的名字创建一个目录，如这里的`overlay2`。

```shell
[root@iZ2zefmrr626i66omb40ryZ docker]$ cd image/
[root@iZ2zefmrr626i66omb40ryZ image]$ ls
overlay2
# 看下里面有哪些文件
[root@iZ2zefmrr626i66omb40ryZ image]$ tree -L 2 overlay2/
overlay2/
├── distribution
│   ├── diffid-by-digest
│   └── v2metadata-by-diffid
├── imagedb
│   ├── content
│   └── metadata
├── layerdb
│   ├── mounts
│   ├── sha256
│   └── tmp
└── repositories.json
```

这里的关键地方是`imagedb`和`layerdb`目录，看这个目录名字，很明显就是专门用来存储元数据的地方。

* layerdb：docker image layer 信息
* imagedb：docker image 信息

因为 docker image 是由 layer 组成的，而 layer 也已复用，所以分成了 layerdb 和 imagedb。

先去 imagedb 看下刚才构建的镜像：

```shell
$  cd overlay2/imagedb/content/sha256
$ ls
[root@iZ2zefmrr626i66omb40ryZ sha256]# ls
0c7ea9afc0b18a08b8d6a660e089da618541f9aa81ac760bd905bb802b05d8d5  61ad638751093d94c7878b17eee862348aa9fc5b705419b805f506d51b9882e7 
// .... 省略
b20b605ed599feb3c4757d716a27b6d3c689637430e18d823391e56aa61ecf01
60d84e80b842651a56cd4187669dc1efb5b1fe86b90f69ed24b52c37ba110aba  ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1
```

可以看到，都是 64 位的ID，这些就是具体镜像信息，刚才构建的镜像ID为`290d8cc1f75a`,所以就找`290d8cc1f75a`开头的文件：

```shell
[root@iZ2zefmrr626i66omb40ryZ sha256]$ cat 290d8cc1f75a4e230d645bf03c49bbb826f17d1025ec91a1eb115012b32d1ff8 
{"architecture":"amd64","config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["bash"],"Image":"sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":null},"container":"ee79bb9802d0ff311de6d606fad35fa7e9ab0c1cb4113837a50571e79c9454df","container_config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh","-c","echo \"Hello world\" \u003e /tmp/newfile"],"Image":"sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":null},"created":"2022-01-17T12:25:14.91890037Z","docker_version":"20.10.6","history":[{"created":"2021-10-16T00:37:47.226745473Z","created_by":"/bin/sh -c #(nop) ADD file:5d68d27cc15a80653c93d3a0b262a28112d47a46326ff5fc2dfbf7fa3b9a0ce8 in / "},{"created":"2021-10-16T00:37:47.578710012Z","created_by":"/bin/sh -c #(nop)  CMD [\"bash\"]","empty_layer":true},{"created":"2022-01-17T12:25:14.91890037Z","created_by":"/bin/sh -c echo \"Hello world\" \u003e /tmp/newfile"}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b","sha256:b3cce2ce0405ffbb4971b872588c5b7fc840514b807f18047bf7d486af79884c"]}}
```

这就是 image 的 metadata，这里主要关注 rootfs：

```shell
# 和 docker inspect 命令显示的内容差不多
// ...
"rootfs":{"type":"layers","diff_ids":
[
"sha256:9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b",
"sha256:b3cce2ce0405ffbb4971b872588c5b7fc840514b807f18047bf7d486af79884c"
]
}
// ...
```

可以看到rootfs的diff_ids是一个包含了两个元素的数组，这两个元素就是组成hello-ubuntu镜像的两个Layer的**`diffID`**。从上往下看，就是底层到顶层，也就是说`9f54eef412...`是image的最底层。

然后根据 layerID 去`layerdb`目录寻找对应的 layer：

```shell
[root@iZ2zefmrr626i66omb40ryZ overlay2]# tree -L 2 layerdb/
layerdb/
├── mounts
├── sha256
└── tmp
```

在这里我们只管`mounts`和`sha256`两个目录，先打印以下 sha256 目录

```shell
$ cd /var/lib/docker/image/overlay2/layerdb/sha256/
$ ls
05dd34c0b83038031c0beac0b55e00f369c2d6c67aed11ad1aadf7fe91fbecda  
// ... 省略
6aa07175d1ac03e27c9dd42373c224e617897a83673aa03a2dd5fb4fd58d589f  
```

可以看到，layer 里也是 64位随机ID构成的目录，找到刚才hello-ubuntu镜像的最底层layer：

```shell
$ cd 9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b
[root@iZ2zefmrr626i66omb40ryZ 9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b]$ ls
cache-id  diff  size  tar-split.json.gz
```

文件含义如下：

* cache-id：为具体`/var/lib/docker/overlay2/<cache-id>`存储路径
* diff：diffID，用于计算 ChainID
* size：当前 layer 的大小



docker使用了chainID的方式来保存layer，layer.ChainID只用本地，根据layer.DiffID计算，并用于layerdb的目录名称。

chainID唯一标识了一组（像糖葫芦一样的串的底层）diffID的hash值，包含了这一层和它的父层(底层)，

* 当然这个糖葫芦可以有一颗山楂，也就是chainID(layer0)==diffID(layer0)；
* 对于多颗山楂的糖葫芦，ChainID(layerN) = SHA256hex(ChainID(layerN-1) + " " + DiffID(layerN))。

```shell
# 查看 diffID，
$ cat diff 
sha256:9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b
```

由于这是 layer0,所以 chainID 就是 diffID，然后开始计算 layer1 的 chainID：

```shell
ChainID(layer1) = SHA256hex(ChainID(layer0) + " " + DiffID(layer1))
```

layer0的 chainID就是`9f54...`,而 layer1的 diffID 根据 rootfs 中的数组可知，为`b3cce...`

计算ChainID：

```shell
$ echo -n "sha256:9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b sha256:b3cce2ce0405ffbb4971b872588c5b7fc840514b807f18047bf7d486af79884c" | sha256sum| awk '{print $1}'
6613b10b697b0a267c9573ee23e54c0373ccf72e7991cf4479bd0b66609a631c
```

> 一定注意要加上 “sha256:”和中间的空格“ ” 这两部分。

因此 layer1的 chainID 就是`6613...`

```shell
$ cd /var/lib/docker/image/overlay2/layerdb/sha2566613b10b697b0a267c9573ee23e54c0373ccf72e7991cf4479bd0b66609a631c
# 根据这个大小可以知道，就是hello-ubuntu 镜像的最上面层 layer
[root@iZ2zefmrr626i66omb40ryZ 6613b10b697b0a267c9573ee23e54c0373ccf72e7991cf4479bd0b66609a631c]$ cat size 
12
# 查看 cache-id 找到 文件系统中的具体位置
[root@iZ2zefmrr626i66omb40ryZ 6613b10b697b0a267c9573ee23e54c0373ccf72e7991cf4479bd0b66609a631c]$ cat cache-id 
83b569c0f5de093192944931e4f41dafb2d7f80eae97e4bd62425c20e2079f65
```

进入具体目录：



```shell
# 进入刚才生成的目录
$ cd /var/lib/docker/overlay2/83b569c0f5de093192944931e4f41dafb2d7f80eae97e4bd62425c20e2079f65
[root@iZ2zefmrr626i66omb40ryZ 83b569c0f5de093192944931e4f41dafb2d7f80eae97e4bd62425c20e2079f65]# ls -al
total 24
drwx-----x  4 root root    55 Jan 17 20:25 .
drwx-----x 53 root root 12288 Jan 17 20:25 ..
drwxr-xr-x  3 root root    17 Jan 17 20:25 diff
-rw-r--r--  1 root root    26 Jan 17 20:25 link
-rw-r--r--  1 root root    28 Jan 17 20:25 lower
drwx------  2 root root     6 Jan 17 20:25 work
# 查看 diff 目录
[root@iZ2zefmrr626i66omb40ryZ 
83b569c0f5de093192944931e4f41dafb2d7f80eae97e4bd62425c20e2079f65]$ cd diff/
[root@iZ2zefmrr626i66omb40ryZ diff]$ ls
tmp
[root@iZ2zefmrr626i66omb40ryZ diff]$ cd tmp/
[root@iZ2zefmrr626i66omb40ryZ tmp]$ ls
newfile
[root@iZ2zefmrr626i66omb40ryZ tmp]# cat newfile 
Hello world
```

可以看到，我们新增的 newfile 就在这里。





## 4. 参考

[a practical look into overlayfs](https://ops.tips/notes/practical-look-into-overlayfs/)

[overlayfs.txt](https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt)

[docker-overlay2文件系统](https://www.jianshu.com/p/3826859a6d6e)