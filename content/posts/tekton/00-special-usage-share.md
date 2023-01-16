---
title: "Tekton教程(四)---实用姿势：Tekton 特殊用法"
description: "一些比较实用的 Tekton 用法分享"
date: 2023-01-14
draft: true
categories: ["tekton"]
tags: ["tekton"]
---

本文为一些比较实用的 Tekton 用法分享， 比如 cronjob + kubectl 命令定时创建 pipelinerun 来运行某个流水线。

<!--more-->



## 1. 实现定时任务

上一篇文章中我们通过 Trigger 组件已经可以实现通过外部事件触发流水线了，但是有部分情况下某个任务可能除了外部触发之外还需要定时触发，这个时候就需要借助别的工具了。



实际上除了 push 代码时需要通过 webhook 触发某些任务外，可能还有一些定时任务，比如每天完成自动构建、自动部署等。

### cronjob + kubectl

这部分我们可以借助 k8 中的 cronjob 来实现，定时创建一个 pipelinerun 即可,就像这样：

```YAML
# cronjob to create create-github-minimal pipeline runs
apiVersion: batch/v1
kind: CronJob
metadata:
  labels:
    app: tekton-pipelinerun-creater
  name: pipelinerun-creater-kubectl
spec:
  concurrencyPolicy: Forbid
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
            - command:
                - /bin/bash
                - -c
                - |
                  cat <<EOF | kubectl create -f -
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
                  EOF
              image: docker.io/alpine/k8s:1.20.7
              imagePullPolicy: IfNotPresent
              name: kubectl
          serviceAccount: tekton-pipelinerun-creater
          restartPolicy: OnFailure
  schedule: 0/1 * * * *
  successfulJobsHistoryLimit: 3
  suspend: false
```

定时任务里通过 kubectl 创建一个pipelinerun 对象以触发对应 pipeline，由于需要创建资源因此需要指定一个有创建pipelinerun 资源权限的 serviceAccount，否则会没有权限从而创建失败。



为 tekton-pipelinerun-creater 赋权限的相关 yaml 如下：

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-pipelinerun-creater
  labels:
    app: tekton-pipelinerun-creater
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-pipelinerun-creater
  labels:
    app: tekton-pipelinerun-creater
rules:
  - apiGroups:
      - tekton.dev
    resources:
      - pipelineruns
    verbs:
      - create
      - get
      - watch
      - list
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-pipelinerun-creater
  labels:
    app: tekton-pipelinerun-creater
roleRef:
  kind: Role
  name: tekton-pipelinerun-creater
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: tekton-pipelinerun-creater
```



### Cronjob +curl

如果创建了 Trigger，我们也可以通过 cronjob 使用 curl 调 eventListenr 的接口来触发事件，然后剩下的工作就交给 Trigger 来完成了。

```YAML
# cronjob to create pipelinerun
apiVersion: batch/v1
kind: CronJob
metadata:
  labels:
    app: tekton-pipelinerun-creater
  name: pipelinerun-creater-curl
spec:
  concurrencyPolicy: Forbid
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
            - command:
                - "sh"
                - "-c"
                - |
                  curl http://el-trigger-pipeline-eventlistener:8080 -d '{"username":"17x"}'
              image: curlimages/curl
              imagePullPolicy: IfNotPresent
              name: kubectl
          restartPolicy: OnFailure
  schedule: 0/1 * * * *
  successfulJobsHistoryLimit: 3
  suspend: false
```

### 小结

Cronjonb + kubectl 方案比较适合只需要定时触发的任务。

Cronjob + curl 方案则比较适合既要根据外部事件触发又要定时触发的任务。



## 2. 垃圾清理

Tekton 运行时间长之后会在 k8s 里累积大量的 pod、pipelinerun 等资源，我们同样可以使用一个 cronjob 来对这些资源做一个清理工作。

```YAML
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-pipelinerun-cleaner
  labels:
    app: tekton-pipelinerun-cleaner
    app.kubernetes.io/name: tekton-pipelinerun-cleaner
    app.kubernetes.io/component: pipelinerun-cleaner
    app.kubernetes.io/part-of: tekton
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-pipelinerun-cleaner
  labels:
    app: tekton-pipelinerun-cleaner
    app.kubernetes.io/name: tekton-pipelinerun-cleaner
    app.kubernetes.io/component: pipelinerun-cleaner
    app.kubernetes.io/part-of: tekton
rules:
  - apiGroups:
      - tekton.dev
    resources:
      - pipelineruns
    verbs:
      - delete
      - get
      - watch
      - list
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-pipelinerun-cleaner
  labels:
    app: tekton-pipelinerun-cleaner
    app.kubernetes.io/name: tekton-pipelinerun-cleaner
    app.kubernetes.io/component: pipelinerun-cleaner
    app.kubernetes.io/part-of: tekton
roleRef:
  kind: Role
  name: tekton-pipelinerun-cleaner
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: tekton-pipelinerun-cleaner
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: tekton-pipelinerun-cleaner
  labels:
    app: tekton-pipelinerun-cleaner
    app.kubernetes.io/name: tekton-pipelinerun-cleaner
    app.kubernetes.io/component: pipelinerun-cleaner
    app.kubernetes.io/part-of: tekton
spec:
  schedule: "* */12 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccount: tekton-pipelinerun-cleaner
          containers:
            - name: kubectl
              image: docker.io/alpine/k8s:1.20.7
              env:
                - name: NUM_TO_KEEP
                  value: "5"
              command:
                - /bin/bash
                - -c
                - >
                  while read -r PIPELINE; do
                    while read -r PIPELINE_TO_REMOVE; do
                      test -n "${PIPELINE_TO_REMOVE}" || continue;
                      kubectl delete ${PIPELINE_TO_REMOVE} \
                          && echo "$(date -Is) PipelineRun ${PIPELINE_TO_REMOVE} deleted." \
                          || echo "$(date -Is) Unable to delete PipelineRun ${PIPELINE_TO_REMOVE}.";
                    done < <(kubectl get pipelinerun -l tekton.dev/pipeline=${PIPELINE} --sort-by=.metadata.creationTimestamp -o name | head -n -${NUM_TO_KEEP});
                  done < <(kubectl get pipelinerun -o go-template='{{range .items}}{{index .metadata.labels "tekton.dev/pipeline"}}{{"\n"}}{{end}}' | uniq);
              resources:
                requests:
                  cpu: 50m
                  memory: 32Mi
                limits:
                  cpu: 100m
                  memory: 64Mi
```



