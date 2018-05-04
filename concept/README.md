# 基本概念
参照kubernetes官方文档。本文中列出的基本概念是最核心的，也是实际项目中最经常用到的

## Overview
1. kubernetes对象
[https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
2. namespace命名空间
[https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
3. label and selector，标签和选择器
[https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
4. kubectl 操作kubernetes对象的方式
[https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/)

## 架构
1. node节点
[https://kubernetes.io/docs/concepts/architecture/nodes/](https://kubernetes.io/docs/concepts/architecture/nodes/)

## 工作负载
1. Pod（很重要）
- [pod overview](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)
- [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
- [pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

2. Deployment
[https://kubernetes.io/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

## 配置
1. confgmap
[https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
2. best practice
[https://kubernetes.io/docs/concepts/configuration/overview/](https://kubernetes.io/docs/concepts/configuration/overview/)

## 服务、负载均衡和网络
1. service
[https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
2. DNS for service
[https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
3. service访问外网
[https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)

## 存储
1 volumes
[https://kubernetes.io/docs/concepts/storage/volumes/](https://kubernetes.io/docs/concepts/storage/volumes/)