---
title: "Tekton教程(二)---构建流水线：Task & Pipeline 基本使用"
description: "tekton 中 task 和 pipeline 具体使用说明"
date: 2023-01-08
draft: false
categories: ["tekton"]
tags: ["tekton"]
---

本文主要记录云原生的 CI/CD 框架 Tekton 中的 task 和 pipeline 资源对象的基本使用，通过 task 构建基本任务单元，然后使用 pipeline 将多个任务组合构建成一个完整的流水线，最后在通过 pipelinerun 触发相应任务。

<!--more-->

在前一篇文章：[Tekton教程(一)---云原生 CICD: Tekton 初体验](https://lixueduan.com/posts/tekton/01-deploy-tekton/) 的最后我们运行了一个简单的 demo，创建了一个简单的 task 并通过 taskrun 进行触发，最后查看 od 日志观察运行结果。对于初次接触 Tekton 的用户可能一时不能理解，不过相信看到这篇文章之后就能够对 Tekton 整个流水线的定义与构建有相应的认识了。



tekton 流水线的构建包括以下两个核心对象的使用：

- 1）**task**：task 对象是 tekton 中任务的最小单位
  - taskrun：task 的关联对象，创建一个 task 之后并不会真的运行，需要使用 taskrun 对象来真正执行。
- 2）**pipeline**：pipeline 则是多个 task 的组合
  - pipelinerun：同 task，pipeline 也需要创建 pipelinerun 对象才会执行。



## 1. task & taskrun

**Task 用于定义具体的任务**，task 是 tekton 中描述任务的最小单位。

> 一个 task 最好只做一件事，这样能更好的复用 task。

一个 task 可以做这些事情：

- 打印一句话
- 调用一次 http 请求
- 克隆一个镜像仓库
- 编译一个 go 程序
- Build 一个镜像
- ...

**Taskrun 用于触发具体的任务。**

Taskrun 有两个作用：

1. 触发一次任务
2. 为任务提供具体参数



### 简单的 task 定义

一个简单的 task   完整 yaml 如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello World"
```

核心部分为 spec.steps，一个任务可以由多个步骤组成。

上述 task 对应的 taskrun 完整 yaml 如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: hello-task-run-
spec:
  taskRef:
    name: hello
```

> 可以看到 taskrun 中通过 **spec.taskRef.name** 来关联 task



#### generateName

需要注意的是 taskrun 这里我们没有指定 name，而是用的 generateName，这样在创建该对象时 k8s 会自动以generateName 为前缀生成该对象的完整名字。

**同时该对象也必须使用 create 命令来创建，而不是 apply。**

**为什么需要用 generateName？**

因为一个 taskrun 只能触发一次任务运行，而一个任务我们可能会反复运行，如果我们在 taskrun 中写死一个名字，就会导致该任务只会触发一次，就算 apply 多次都会因为内容没有任何变化而直接被忽略掉。



### 带参数的 task 定义

Task 中除了运行固定脚本、命令之外，还可以从外部传递参数，就像这样：

一个带参数的 task 定义如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  params:
  - name: username
    type: string
  steps:
    - name: echo
      image: alpine
      script: |
        #!/bin/sh
        echo "$(params.username)"
```

这个 task 定义中增加了 **spec.params** 字段用于声明该任务需要哪些参数，然后在 steps 中使用 **$(params.paramsName)** 语法获取对应参数。

该任务对应的 taskrun 定义如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: hello-task-run-
spec:
  taskRef:
    name: hello
  params:
    - name: username
      value: "17x"
```

可以看到除了最基本的  spec.taskRef 之外新增了一个 spec.params 字段用于定义 task 中用到的参数的具体值。

> 即：task 中最终这个参数的值则是通过 taskrun 指定。



### Demo

跑一个简单的 task 来体验一下。



#### Create task

首先创建一个简单的 task,hello-task.yaml 内容如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  params:
  - name: username
    type: string
  steps:
    - name: echo
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello $(params.username)"
```

 apply 到集群里

```Bash
kubectl apply -f hello-task.yaml
```

查看创建结果

```Bash
[root@caas-console ~]# kubectl get task
NAME    AGE
hello   6m15s
```



#### Create taskRun

然后创建一个 taskrun 来触发该任务，同时提供 task 需要的参数，hello-taskrun.yaml 完整内容如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: hello-task-run-
spec:
  taskRef:
    name: hello
  params:
    - name: username
      value: "17x"
kubectl create -f hello-taskrun.yaml 
```

查看创建结果

```Bash
[root@caas-console ~]# kubectl get taskrun
NAME                   SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-task-run-fg6k8   Unknown     Running     8s
```

可以看到 taskrun 创建成功，由于还在运行中，所以 SUCCEEDED 字段为 Unknown，等运行完成后就会根据结果改为 true 或者 false。



#### 查看运行结果

同时查看 pod 可以看到为这个任务启动了一个 pod 来运行，由于任务比较简单所以已经运行完成了。

```Bash
[root@caas-console ~]# kubectl get po
NAME                       READY   STATUS      RESTARTS   AGE
hello-task-run-fg6k8-pod   0/1     Completed   0          79s
```

我们可以看下这个 pod 的日志，是否打印出了 "Hello 17x" 这句话。

```Bash
[root@caas-console ~]# kubectl logs hello-task-run-fg6k8-pod
Hello 17x
```

果然打印出了 "Hello 17x",说明任务运行成功。

至此 task 相关教程就结束了，除了简单的执行一个脚本之外 task 还能做很多事情，就交给大家自行探索了。



## 2. pipeline & pipelinerun

**Pipeline 翻译为流水线，它可以看做是多个 task 的组合。**

一般我们的任务都比较复杂，由多个步骤组成，虽然可以把每个步骤对应为 task 中的一个 step，但是这样不利于 task 复用，所以一般不这样做。

推荐用法：**将比较通用的步骤单独定义为 task，然后使用 pipeline 将多个 task 编排为一个流水线** ，这样既能使用功能，也更利于 task 复用。

比如一个构建镜像的 pipeline 就包含两个步骤：

1. Git clone 拉取代码
2. 构建镜像并上传

可以直接在一个 task 里写两个 step，但是这两个步骤都是比较通用的，因此最好是单独定义成 task。

**pipelinerun 和 taskrun 基本一致，用于触发流水线以及为 pipeline 提供必要的参数。**

> pipelinerun 之于 pipelinerun 等于 taskrun 之于 task。



### pipeline 定义

一个简单的 pipeline 如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-goodbye
spec:
  params:
  - name: username
    type: string
  tasks:
    - name: hello
      taskRef:
        name: hello
    - name: goodbye
      runAfter:
        - hello
      taskRef:
        name: goodbye
      params:
      - name: username
        value: $(params.username)
```

可以看到，pipeline 中通过 **spec.tasks** 指定多个 task，每个 task 里通过 **taskRef.name** 关联到具体的 task 实例。

由于某个task 是需要参数的，因此需要在 pipeline 的 spec.params 里定义好需要的参数，然后在 spec.tasks 里也需要再次定义 params，不过 task 里直接通过 **$(params.username)** 获取具体值**。**

**注意**：需要注意的是，tasks 中的**任务不保证先后顺序**，因此如果不同任务之间有依赖关系可以使用 **runAfter** 字段来指定先后关系。



### pipelinerun 定义

之前 pipeline 对应的 pipelinerun 定义如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: hello-goodbye-run
spec:
  pipelineRef:
    name: hello-goodbye
  params:
  - name: username
    value: "Tekton"
```

可以看到 pipelinerun 中通过 **spec.pipelineRef.name** 来关联到具体 pipeline，同时通过 **spec.params** 指定具体参数的值。

> 和 taskrun 类似



### Demo

#### Create goodbye task

Pipeline 一般需要多个task，因此我们再创建一个 task，goodbye-task.yaml 完成内容如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: goodbye
spec:
  params:
  - name: username
    type: string
  steps:
    - name: goodbye
      image: ubuntu
      script: |
        #!/bin/bash
        echo "Goodbye $(params.username)!"
kubectl apply -f goodbye-task.yaml
```

查看一下当前定义好的 task：

```Bash
[root@caas-console ~]# kubectl get task
NAME      AGE
goodbye   20s
hello     32m
```

可以看到已经有两个 task 了，可以编排 pipeline 了。



#### Create pipeline

使用 hello 和 goodbye 两个 task 来构建一个 pipeline，hello-goodbye-pipeline.yaml 完成内容如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-goodbye
spec:
  params:
  - name: username
    type: string
  tasks:
    - name: hello
      taskRef:
        name: hello
      params:
      - name: username
        value: $(params.username)
    - name: goodbye
      runAfter:
        - hello
      taskRef:
        name: goodbye
      params:
      - name: username
        value: $(params.username)
```

由于两个 task 都需要参数，因此两个 task 里都需要增加 params 字段。

```Bash
kubectl apply -f hello-goodbye-pipeline.yaml 
```

查看创建好的 pipeline

```Bash
[root@caas-console ~]# kubectl get pipeline
NAME            AGE
hello-goodbye   5s
```



#### Create pipelinerun

然后创建 pipelinerun 来真正执行这个 pipeline，hello-goodbye-run.yaml 完整内容如下：

```YAML
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: hello-goodbye-run
spec:
  pipelineRef:
    name: hello-goodbye
  params:
  - name: username
    value: "Tekton"
```

这里我们把 username 参数改成 tekton。

```Bash
kubectl create -f hello-goodbye-run.yaml 
```

查看状态

```Bash
[root@caas-console ~]# kubectl get pipelinerun
NAME                     SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
hello-goodbye-runqrpch   Unknown     Running   6s
```



#### 查看运行结果

查看具体的 pod

```Bash
NAME                                 READY   STATUS      RESTARTS   AGE
hello-goodbye-runqrpch-goodbye-pod   0/1     Completed   0          34s
hello-goodbye-runqrpch-hello-pod     0/1     Completed   0          43s
```

可以看到，hello-pod 和 goodbye-pod  都已经是 Completed 状态了，说明 pipeline 已经执行完毕，再次查看 pipelinerun 的状态

```Bash
[root@caas-console ~]# kubectl get pipelinerun
NAME                     SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-goodbye-runqrpch   True        Succeeded   87s         49s
```

SUCCEEDED 字段已经变成 True 了。

分别查看两个 pod 的日志，确实下是否真的执行成功

```Bash
[root@caas-console ~]# kubectl logs hello-goodbye-runqrpch-hello-pod
Hello Tekton
[root@caas-console ~]# kubectl logs hello-goodbye-runqrpch-goodbye-pod
Goodbye Tekton!
```

确实是按照定义输出了对应内容，说明 pipeline 真的执行完成了。

至此，tekton pipeline 相关教程就结束了，后续进阶的话就是定义的 pipeline 更加复杂罢了。



## 3. 小结

Tekton 流水线构建主要分为以下两个部分：

- Task & taskrun
  - task：定义一个任务
  - taskrun：触发指定任务并传递参数
- Pipeline & pipelinerun
  - pipeline：编排多个任务，组成一个流水线以实现某个功能
  - pipelinerun：类似于 taskrun，触发指定 pipeline 并传递参数


一句话描述：**通过 task 定义基本任务单元，通过 pipeline 将多个任务进行组合，构建出一个能实现某个完整功能的流水线，最后通过 pipelinerun 触发对应流水线的运行。**



