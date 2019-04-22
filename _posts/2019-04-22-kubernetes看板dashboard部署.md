---
layout:     post
title:      kubernetes资源看板dashboard部署
subtitle:   kubernetes资源看板dashboard部署搭建与访问
date:       2019-04-22
author:     cugblack
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - kubernetes
    - 看板
    - 部署
    
---

> *  在部署了kubernetes集群之后，我们需要一个可视化的资源看板来查看我们集群内部的资源，dashborad是我们目前的选择（虽然各种不好用）

# 部署

## https方式

部署的话有两种方式，一种是采用常规的https（8443端口），在进行资源查看时需要登录认证，否则没有权限；另外一种是采用http方式，比较危险，无需任何授权即可访问或操集群内的资源请谨慎选择。

yaml这里就不再贴了，给个官方连接[部署需要的yaml](https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml)，如果需要外部访问，可以使用ingress来代理dashboard的服务，或者修改service使用`nodePort`。

---

###  访问

现在服务已经正常启动了，你需要从页面上访问UI，这是你还需要创建一个用户，并且给他分配相应的权限，一个大致的yaml如下：

```
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

上面的大致意思是创建一个用户`admin`，并将他绑定到集群管理角色，这意味着他可以操作所有的命名空间里的资源，非常危险，你需要创建几个普通的针对不同命名空间的用户，来保证登录的用户没有超过他权限的授权。创建好之后，你需要执行下面的命令查看这个用户的token:

```
#列出secret
kubectl -n kube-system get secret|grep admin-token
#查看token
kubectl -n kube-system describe secret secret_name
```
然后，你就可以使用这个token登录webUI界面了。

## http部署dashboard [!不推荐!]

> 不开启任何授权，通过dashboard的9090端口访问，任何人都可以对集群内的所有资源进行任意操作！！

修改yaml中的几个部分即可，deployment内容器端口修改`containerPort:9090`，健康检查的端口也需要改为9090，service内的端口映射改为9090，http。




