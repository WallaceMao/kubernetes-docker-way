# DNS服务安装与运行
本文使用[coredns](https://github.com/coredns/coredns)构建。
coredns的运行原理是在master的kube-system空间内构建一个DNS基础服务的cluster，用来做域名解析

## 确定dns-cluster-ip和dns-domain

- 由于coredns是运行在一个cluster上，因此需要确定dns的cluster ip，这个ip必须在apiserver定义的cluster-range内
- 还需要确定一个dns-domain的域名，pod直接通过域名获取服务。一般的域名例如：cluster.local

## 新建DNS相关的ServiceAccount

使用一下account.yaml文件创建：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
```

`kubectl create -f account.yaml`

## 新建配置

新建configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes __PILLAR__DNS__DOMAIN__ in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
        }
```

**注意** 其中`__PILLAR__DNS__DOMAIN__`需要替换成定义的域名，例如:cluster.local

运行命令：
`kubectl create -f configmap.yaml`

## 部署

新建deployment.yaml

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
spec:
  serviceAccountName: coredns
  tolerations:
    - key: node-role.kubernetes.io/master
      effect: NoSchedule
    - key: "CriticalAddonsOnly"
      operator: "Exists"
  containers:
  - name: coredns
    image: coredns/coredns:1.1.0
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        memory: 170Mi
      requests:
        cpu: 100m
        memory: 70Mi
    args: [ "-conf", "/etc/coredns/Corefile" ]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/coredns
    ports:
    - containerPort: 53
      name: dns
      protocol: UDP
    - containerPort: 53
      name: dns-tcp
      protocol: TCP
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 60
      timeoutSeconds: 5
      successThreshold: 1
      failureThreshold: 5
  dnsPolicy: Default
  volumes:
    - name: config-volume
      configMap:
        name: coredns
        items:
        - key: Corefile
          path: Corefile
```

运行命令：

`kubectl create -f deployment.yaml`

## 新建服务

新建service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: __CLUSTER__IP__
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

**注意**__CLUSTER__IP__需要修改为分配的地址

执行命令：

`kubectl create -f service.yaml`

## 修改kubelet

编辑 minion中的/etc/kubernetes/kubelet文件，添加：

`KUBELET_ARGS="--cluster-dns=<cluster-dns> --cluster-domain=<cluster-domain>"`

其中:
- <cluster-dns>为dns service 的cluster ip，与configmap.yaml中的__PILLAR__DNS__DOMAIN__一致。例如：10.0.0.2
- <cluster-domain>是设置的cluster域名，与service.yaml中的__CLUSTER__IP__一致。例如：cluster.local

修改后，重启kubelet

```
systemctl daemon-reload
systemctl restart kubelet
```

## 测试是否成功

新建pod，并在pod内检查 pod-ip-address.my-namespace.pod.cluster.local 是否解析成功

DNS详细参考：[https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)