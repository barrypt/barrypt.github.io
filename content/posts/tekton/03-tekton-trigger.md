---
title: "Tekton教程(三)---解放双手：使用 Trigger 自动触发流水线"
description: "借助 Trigger 组件实现通过外部事件触发指定流水线"
date: 2023-01-15
draft: false
categories: ["tekton"]
tags: ["tekton"]
---

本文主要记录了创建 Tekton pipeline 之后，如何解放双手，借助 Trigger 组件实现通过外部事件（比如代码仓库的 commit 事件）触发指定流水线，而不是手动创建 pipelinerun 来触发。

<!--more-->



## 1. Trigger 是什么？

根据前面 task 和 pipeline 的教程 [Tekton教程(二)---构建流水线：Task & Pipeline 基本使用](https://lixueduan.com/posts/tekton/02-tekton-task-pipeline/) 可以知道，想要触发某个 task 或者 pipeline 必须手动创建一个 taskrun 或者 pipelinerun，可以说是自动化了但又没完全自动化。

今天介绍的组件 Trigger 则可以实现通过外部事件来触发对应的任务，比如有代码提交时触发自动构建以及部署任务。

Trigger 组件就是用来解决这个触发问题的，**它可以从各种来源的事件中检测并提取需要信息，然后根据这些信息来创建 TaskRun 和 PipelineRun**，还可以将提取出来的信息传递给它们以满足不同的运行要求。

> gitlab、github 的 webhook 就是一种最常用的外部事件，通过 Trigger 组件就监听这部分事件从而实现在提交代码后自动运行某些任务。



### 核心模块

其核心模块如下：

- **EventListener**：时间监听器，是外部事件的入口 ，通常需要通过HTTP方式暴露，以便于外部事件推送，比如配置Gitlab的Webhook。
- **Trigger**：指定当 EventListener 检测到事件发生时会发生什么，它会定义 TriggerBinding、TriggerTemplate 以及可选的 Interceptor。
- **TriggerTemplate**：用于模板化资源，根据传入的参数实例化 Tekton 对象资源，比如 TaskRun、PipelineRun等。
- **TriggerBinding**：用于捕获事件中的字段并将其存储为参数，然后会将参数传递给 TriggerTemplate。
- **ClusterTriggerBinding**：和 TriggerBinding 相似，用于提取事件字段，不过它是集群级别的对象。
- **Interceptors**：拦截器，在 TriggerBinding 之前运行，用于负载过滤、验证、转换等处理，只有通过拦截器的数据才会传递给TriggerBinding。



### 工作流程

具体工作流程如下：

![tekton-dashboard.png](../../../img/tekton/trigger/TriggerFlow.svg)

* EventListener 用于监听外部事件（具体触发方式为 http），外部事件产生后被 EventListener 捕获，然后进入处理过程。

* 首先会由 Interceptors 来进行处理（如果有配置 interceptor 的话），对负载过滤、验证、转换等处理，类似与 http 中的 middleware。

* Interceptors 处理完成后无效的事件就会被直接丢弃，剩下的有效事件则交给 TriggerBinding 处理，
* TriggerBinding 实际上就是从事件内容中提取对应参数，然后将参数传递给 TriggerTemplate。

* TriggerTemplate 则根据预先定义的模版以及收到的参数创建 TaskRun 或者 PipelineRun 对象。
* TaskRun 或者 PipelineRun 对象创建之后就会触发对应 task 或者 pipeline 运行，整个流程就全自动了。



## 2. Trigger 组件中的模块

### EventListener

一个简单的 EventListener 定义如下：

```YAML
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: trigger-pipeline-eventlistener
spec:
  resources:
    kubernetesResource:
      serviceType: NodePort
  serviceAccountName: tekton-triggers-gitlab-sa
  triggers:
    - bindings:
        - ref: tr-pipeline-binding
      template:
        ref: tr-pipeline-template
```

参数详解：

- 首先 spec.resources.kubernetesResource.serviceType 定义了这个 EventListener 接收外部事件的 svc 的类型，这里选择 nodePort 编译外部调用。
  - **外部事件的触发都是通过 http 调用实现的，也就是说这个 EventListener 通过 svc 暴露出去一个接口，只要调用该接口就算是触发了一个事件。**
- 然后 spec.serviceAccountName 则是指定了这个 EventListener 使用的 serviceAccount，因为 EventListener 最终会创建 taskrun、pipelinerun 同时会查询一些其他信息，因此需要为其配置一个 serviceAccount，同时还需要为这个 serviceAccount 赋予相应的权限。
- 最后 spec.triggers.bindings 就是关联到这个 EventListener 的 TriggerBinding，而 spec.triggers.template 则是关联的 TriggerTemplate。
  - 二者都是通过名字进行关联。

参数分析后，这个 EventListener 的含义就比较明显了，任何由该 EventListener 监听的事件都交给  tr-pipeline-binding 这个 TriggerBinding 来解析参数，然后将参数传递给 tr-pipeline-template 这个 TriggerTemplate 创建对应 taskrun 或者 pipelinerun。过程中需要 的权限由 tekton-triggers-gitlab-sa 这个 serviceAccount 提供。同时这个 EventListener 对外暴露的 service 类型为 NodePort。



### TriggerBinding

一个简单的 TriggerBinding 定义如下：

```YAML
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: pipeline-binding
spec:
  params:
  - name: gitrevision
    value: $(body.head_commit.id)
  - name: gitrepositoryurl
    value: $(body.repository.url)
  - name: contenttype
    value: $(header.Content-Type)
```

这个 TriggerBinding 的 spec.params 则定义了该 TriggerBinding 会尝试从事件内容中提取的参数。

spec.params.name 定义了参数名，spec.params.value 则定义了参数的取值方式。

比如参数 gitrevision 的 value 为 $(body.head_commit.id) 就是说去 body.head_commit.id 这个字段的内容作为自己的值。

> Event body 一般是一个 json 格式的数据



### TriggerTemplate

一个简单的 TriggerTemplate 定义如下：

```YAML
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-template
spec:
  params:
  - name: gitrevision
    description: The git revision
    default: main
  - name: gitrepositoryurl
    description: The git repository url
  - name: message
    description: The message to print
    default: This is the default message
  - name: contenttype
    description: The Content-Type of the event
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: simple-pipeline-run-
    spec:
      pipelineRef:
        name: simple-pipeline
      params:
      - name: message
        value: $(tt.params.message)
      - name: contenttype
        value: $(tt.params.contenttype)
      - name: git-revision
        value: $(tt.params.gitrevision)
      - name: git-url
        value: $(tt.params.gitrepositoryurl)
      workspaces:
      - name: git-source
        emptyDir: {}
```

相信看到这个 yaml 定义基本能猜到具体是做什么的，spec.params 定义了这个 TriggerTemplate 需要哪些参数，需要注意的是 **TriggerTemplate 和 TriggerBinding 中的参数是通过名字关联的**，也就是说这里的参数名必须和 TriggerBinding 里定义的参数名一致才能对应上。

spec.resourcetemplates 则定义了这个 TriggerTemplate 最终会创建的资源模版，比如这里就会创建一个 PipelineRun，然后 pipelinerun 可以从 TriggerTemplate 中获取参数，比如 **$(tt.params.gitrepositoryurl)**

就是获取 gitrepositoryurl 这个参数，这里的 tt 应该就是 TriggerTemplate 的缩写。

到这里整个参数的传递流程就清晰了：

1. PipelineRun 的参数也是从 TriggerTemplate 中取的
2. TriggerTemplate 中的参数值来源于 TriggerBinding
3. TriggerBinding 中的参数则是从 event body 中提取到的

也就是说最终生成的 pipelinerun 要什么参数在触发对应事件时就要相应的携带过来，然后经过 TriggerBinding、TriggerTemplate 最终进入到 pipelinerun，然后进入到 pipeline，最后进入到对应的 task。



### RBAC

这里为 EventListener 中使用的 serviceAccout 赋予权限。

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-triggers-gitlab-sa
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-triggers-gitlab
rules:
  # Permissions for every EventListener deployment to function
  - apiGroups: ["triggers.tekton.dev"]
    resources: ["eventlisteners", "triggerbindings", "triggertemplates","clustertriggerbindings", "clusterinterceptors","interceptors","triggers"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    # secrets are only needed for Github/Gitlab interceptors, serviceaccounts only for per trigger authorization
    resources: ["configmaps", "secrets", "serviceaccounts"]
    verbs: ["get", "list", "watch"]
  # Permissions to create resources in associated TriggerTemplates
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns", "pipelineresources", "taskruns"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-triggers-gitlab-binding
subjects:
  - kind: ServiceAccount
    name: tekton-triggers-gitlab-sa
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-gitlab
```





## 3. Demo

### 创建 Trigger 组件

创建一个简单的 Trigger 进行测试，hello-trigger.yaml 完整内容如下：

```YAML
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: trigger-pipeline-eventlistener
spec:
  resources:
    kubernetesResource:
      serviceType: NodePort
  serviceAccountName: tekton-triggers-gitlab-sa
  triggers:
    - bindings:
        - ref: tr-pipeline-binding
      template:
        ref: tr-pipeline-template
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: tr-pipeline-binding
spec:
  params:
    - name: username
      value: $(body.username)
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: tr-pipeline-template
spec:
  params:
    - name: username
      description: user for say hello
      default: tekton
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: hello-goodbye-
      spec:
        params:
        - name: username
          value: $(tt.params.username)
        pipelineRef:
          name: hello-goodbye
```

注意：TriggerTemplate 里的 pipeline 也要使用 generateName，否则名字相同、内容也相同的 pipelinerun 就会被忽略掉。

Apply 到 k8s

```Bash
kubectl apply -f hello-trigger.yaml
```

查看创建好的 eventlistener 及其 svc

```Bash
[root@caas-console ~]# kubectl get eventlistener
NAME                             ADDRESS                                                                   AVAILABLE   REASON                       READY   REASON
trigger-pipeline-eventlistener   http://el-trigger-pipeline-eventlistener.default.svc.cluster.local:8080   False       MinimumReplicasUnavailable   False
[root@caas-console ~]# kubectl get svc
NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
el-trigger-pipeline-eventlistener   NodePort    10.108.222.103   <none>        8080:30262/TCP,9000:30330/TCP   2m2s
```

可以看到确实创建了一个 NodePort 类型的 svc，接下来我们只需要调用这个接口即可触发事件。



### 调用接口触发事件

正常情况下一般是把这个 endpoint 配置到 gitlab、github 仓库中作为 webhook，不过手动调用也可以触发，这里简单起见就直接用 curl 来调用。

内网测试，直接通过 svc 的 clusterIP 调用即可

```Bash
curl http://10.108.222.103:8080 -d '{"username":"17x"}'
```

结果如下：

```Bash
[root@caas-console ~]# curl http://10.108.222.103:8080 -d '{"username":"17x"}'
{"eventListener":"trigger-pipeline-eventlistener","namespace":"default","eventListenerUID":"333aff20-2045-43b3-9fe7-78473840aced","eventID":"fd21920b-848a-4f9e-a0b1-b14555ce4136"}
```

可以看到调用成功了，并且返回了这个事件由哪个 eventListener 监听的，具体咋那个 namespace 下，本次事件的 id 等信息。

查看是否创建出了 pipelinerun

```Bash
[root@caas-console ~]# kubectl get pipelinerun
NAME                                                     SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-goodbye-run-50649b48-ddd3-434f-ba70-aa27c87f9342   Unknown     Running     4s
```

Pipelinerun 已经成功创建出来了，说明 Trigger 是正常工作的。

查看任务执行情况

```Bash
[root@tekton ~]# kubectl get po
hello-goodbye-2x2jt-goodbye-pod                                   0/1     Completed   0          9s
hello-goodbye-2x2jt-hello-pod                                     0/1     Completed   0          18s
[root@tekton ~]# kubectl logs hello-goodbye-2x2jt-hello-pod
Hello 17x
[root@tekton ~]# kubectl logs hello-goodbye-2x2jt-goodbye-pod
Goodbye 17x!
```

可以看到两个 task 都已经完成了，而且根据日志打印的 username 为 17x，和我们用 curl 触发事件时传递的参数是一样的，说明 TriggerBinding 以及 TriggerTemplate 都是正常的。

至此，tekton 中的 trigger 组件教程就结束了。



## 4. 小结

本文主要讲了如何配置 Tekton Trigger 组件实现通过外部事件触发流水线。

相关知识点：

1）什么是 Trigger：它可以从各种来源的事件中检测并提取需要信息，然后根据这些信息来创建 TaskRun 和 PipelineRun。

2）Trigger 组件中的模块：包括 EventListener、TriggerBinding、TriggerTemplate、Interceptors 等。

3）Trigger 组件的具体工作流程如下：

- 1）EventListener 通过 svc 方式对外暴露一个接口，
- 2）外部系统调用该接口并传递参数以触发一个事件
- 3）EventListener 收到事件后首先由 Interceptors 进行验证、转换等处理，然后交给 TriggerBinding
- 4）TriggerBinding 从接收到的 event body 中解析出对应参数并传递给 TriggerTemplate
- 5）TriggerTemplate 根据接受到的参数以及模版创建出对应的 taskrun 或者 pipelinerun 等资源，用于触发对应任务。
- 6）Taskrun 或者 pipelinerun 创建后就进入到之前两个模块的范围了，Trigger 的任务就算是完成了。

