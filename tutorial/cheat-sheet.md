# 集群管理
## kubernetes的基本操作

1. 新增kubernetes对象
`kubectl create -f nginx.yaml`

2. 删除kubernetes对象
`kubectl delete -f nginx.yaml`

3. 替换更新kubernetes对象
`kubectl edit [type] [type name]`，例如：`kubectl edit deploy nginx`

4. 查看kubernetes对象

- `kubectl get [type] [type name]`：列出所有当前namespace下的指定type的kubernetes对象，例如：`kubectl get deploy nginx`;
- `kubectl get [type] [type name] -o wide`：列出所有当前namespace下的指定type的kubernetes对象的详扩展信息，例如：`kubectl get deploy nginx -o wide`;
- `kubectl get [type] [type name] -o yaml`：列出所有当前namespace下的指定type的kubernetes对象的详节点状态信息，例如：`kubectl get deploy nginx -o yaml`;
- `kubectl describe [type] [type name]`：列出所有当前namespace下的指定type的kubernetes对象的详节点状态信息，例如：`kubectl get deploy nginx -o yaml`;


## Kubernetes context操作

1. 查看当前的`context`
`kubectl config current-context`

2. 切换`context`

`kubectl config use-context test-admin`

## kubernetes 调试

1. 进入到容器内部
`kubectl exec -it [podName] bash`

2. 查看log日志
- `kubectl log -f --tail=100 [podName]`：滚动查看容器的运行日志
- `kubectl log -p --tail=100 [podName]`：如果容器发生了重启，查看前一个容器的日志

## 集群更新

1. 更新k8s中镜像版本：
`kubectl set image deployment/[deployName] [containerName]=[imagePath]`，例如：`kubectl set image deployment nginx-deploy test-nginx=registry-vpc.cn-beijing.aliyuncs.com/rsq-public/nginx:1.14`

## 集群回滚

1. 查看历史版本
`kubectl rollout history deployment nginx-deploy`

2. 查看某个历史版本的详细信息
`kubectl rollout history deployment nginx-deploy --revision=[revisionNumber]`

3. 将deploy回滚到之前的某个版本
`kubectl rollout undo deployment nginx-deploy --to-revision=[revisionNumber]`

## 集群伸缩

1. 设置集群副本
`kubectl scale deployment nginx-deploy --replicas=1`

## 集群重启节点

1. 抽干节点容器
`kubectl drain [node name]` 

2. 重启节点

3. 将节点加入集群
`kubectl uncordon [node name]`