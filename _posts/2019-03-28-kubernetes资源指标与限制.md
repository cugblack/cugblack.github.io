---
layout:     post
title:      kubernetes资源配额与资源限制
subtitle:   kubernetes资源配额与资源限制-ResourceQuota与LimitRange
date:       2019-03-28
author:     cugblack
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - Kubernetes
    - 资源限制
    - limitRange
    - pod
    - namespace
---

> kubernets集群内，容器与pod消耗或者占据资源的多少，如何合理的分配并使用集群的资源至关重要，这就涉及到了kubernetes内的两个资源指标，`ResourceQuato`与`LimitRange`

---
## 为什么需要资源配额与资源限制

多团队协作的情况下，资源配额很有必要：

>+ 不同团队的命名空间分配不同的资源量，按需分配
>+ 管理员对命名空间进行一种或多种的配额限制
>+ 计算资源的配额被创建后，对应的命名空间将不会创建未设置资源请求或限制的pod


如果你的命名空间有资源配额，那么默认内存限制是很有帮助的：

>+ 运行在命名空间中的每个容器必须有自己的内存限制。
>+ 命名空间中所有容器的内存使用量之和不能超过声明的限制值。

 如果一个容器没有声明自己的内存限制，会被指定默认限制，然后它才会被允许在限定了配额的命名空间中运行


---
## 使用资源限制-resourceQuota

kubernetes内有两种对资源分配管理相关的控制策略插件  `ResourceQuota`和  `LimitRange`

这两种资源都是对命名空间namespace生效，`ResourceQuota` 用来限制 namespace 中所有的 Pod 占用的总的资源 request 和 limit，而 `LimitRange` 是用来设置 namespace 中 Pod或者container 的默认的资源 request 和 limit 值。
 
---


> Kubernetes 的众多发行版本默认开启了资源配额的支持。当在apiserver的`--admission-control`配置中添加`ResourceQuota`参数后，便启用了。 当一个命名空间中含有ResourceQuota对象时，资源配额强制执行。一个命名空间最多只能有一个`ResourceQuota`对象。

---

### 资源配额分为三种类型：

```
    计算资源配额
    存储资源配额
    对象数量配额
```

####计算资源配额

用户可以对指定的命名空间namespace下的*计算资源总量*进行限制

以下是一个计算资源配额的demo：

[计算资源配额](https://github.com/cugblack/k8s/blob/master/templates/resourceQuota.yaml)

    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-resources
      namespace: develop
    spec:
      hard:
        pods: "40"             #命名空间允许创建pod总数
        requests.cpu: "2"      #非终止态的所有pod, cpu请求总量不能超出此值。
        requests.memory: 100Gi #非终止态的所有pod, 内存请求总量不能超出此值。
        limits.cpu: "4"        #非终止态的所有pod， cpu限制总量不能超出此值。
        limits.memory: 200Gi   非终止态的所有pod, 内存限制总量不能超出此值。

---
#### 存储资源配额

用户可以对给定namespace下的 存储资源 总量进行限制。

此外，还可以根据相关的存储类（Storage Class）来限制存储资源的消耗。

支持的类型及指标格式：

    requests.storage              #所有的PVC中，存储资源的需求不能超过该值。
    persistentvolumeclaims        #namespace中所允许的 PVC 总量。
    <storage-class-name>.storageclass.storage.k8s.io/requests.storage

---
#### 对象数量配额

一个给定类型的对象的数量可以被限制，以下是一个对象资源配额限制的demo:

[对象资源配额](https://github.com/cugblack/k8s/blob/master/templates/resourceQuota.yaml)

    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: object-counts
      namespace: develop
    spec:
      hard:
        configmaps: "20"
        persistentvolumeclaims: "8"
        replicationcontrollers: "20"
        secrets: "10"
        services: "20"
        services.loadbalancers: "5"
        
 ---
 
 ### 请求与条件
 
分配计算资源时，每个容器可以为CPU或内存指定请求和约束。 也可以设置两者中的任何一个。

如果配额中指定了 requests.cpu 或 requests.memory 的值，那么它要求每个进来的容器针对这些资源有明确的请求。 如果配额中指定了 limits.cpu 或 limits.memory的值，那么它要求每个进来的容器针对这些资源指定明确的约束。

>如果配置了配额但是创建资源时没有声明对应的限制或者requst，则资源无法创建成功。



## 配置默认namespace资源请求与限制-limitRange


>给命名空间namespace创建默认的request与limit

kubernetes内有一种资源`limitRange`，用来对namespace设置资源请求requests与限制limits。

以下是一个limitRange的demo:
[limitRange.yaml](https://github.com/cugblack/k8s/blob/master/templates/limitRange.yaml)
    
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: mem-limit-range
      namespace: test
    spec:
      limits:
      - default:
          memory: 512Mi    
          cpu: 4
        defaultRequest:
          memory: 256Mi
          cpu: 1
        type: Container # 可以是pod，persistentvolume
 
 通过以上文件，我们对命名空间test设置了默认的资源限制，内存限制为512，默认内存请求`requests.memory`为256，cpu限制为4，默认cpu请求`requests.cpu`为1。
 
 之后如果你在test这个命名空间创建pod时，你会发现以下效果：
 
     创建pod时只设置了limits,而没有设置requests，那么requests值将会设置为与limits值相同。
     创建pod时只设置了requests，没有设置limits，那么limits值将会与你设置的命名空间test的默认请求值`defaultRequest`相同
     
 ---
 
 ## 质量指标Qos(quality of service)
 
 ### 质量指标有三种：
 
     Guareteen ：Pod 里的每个容器都必须有内存限制和请求，而且必须是一样的。
     Burstable ：Pod 不满足 QoS 等级 Guaranteed 的要求，且pod内至少有一个容器有内存或者 CPU 请求。
     BestEffort：Pod 里的容器必须没有任何内存或者 CPU　的限制或请求。
 
 
 ### 资源回收策略
 
 >cpu为可压缩资源，当cpu资源即将耗尽时会进行资源压缩，压缩分配给pod的cpu，但不会kill。
 
 kubernets集群内，当某个节点上的可用资源较少时，即内存或cpu即将耗尽，这时kubelet会执行资源回收策略，Kubernetes通过cgroup给pod设置QoS级别，当资源不足时先kill优先级低的pod，在实际使用过程中，通过OOM分数值来实现，OOM分数值从0-1000。
 
     优先级：
       BestEffort类pod：优先级最低，当内存资源不足时，此类资源最先被杀掉。
       Burstable类pods：系统用完了全部内存，且没有Best-Effort container可以被kill时，该类型pods会被kill掉。
       Guaranteed pods：系统用完了全部内存、且没有Burstable与Best-Effort container可以被kill，该类型的pods会被kill掉。

>注：如果pod进程因使用超过预先设定的limites而非Node资源紧张情况，系统倾向于在其原所在的机器上重启该container或本机或其他重新创建一个pod。


>>[参考官方文档](https://kubernetes.io/docs/concepts/policy/resource-quotas/)