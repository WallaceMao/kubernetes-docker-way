# minion node安装与配置
本章节的操作需要在每一个minion node中进行

## 安装docker
本文通过rpm的方式安装docker，版本是`17.12.1-ce`。
在两台minion机器上，需要分别安装docker。
安装方式可参考[docker官方文档](https://docs.docker.com/install/linux/docker-ce/centos/#docker-ee-customers)

1. 获取docker rpm包

在minion机器上，使用wget下载最新版本的docker rpm包:

    wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-17.12.1.ce-1.el7.centos.x86_64.rpm

2. 运行安装

运行:

    sudo yum install /path/to/package.rpm

3. 设置docker的开机自启动，并启动docker

运行:

    systemctl enable docker
    systemctl start docker

## 安装kubernetes node
kubernetes的安装方式很多，本文下载官方发布的Binary版本的压缩包

1. 下载官方压缩包

在[kubernetes官方发布页面](https://kubernetes.io/docs/imported/release/notes/#node-binaries)找到`Node Binaries`压缩包的下载地址。
使用wget下载

`wget https://dl.k8s.io/v1.10.0/kubernetes-node-linux-amd64.tar.gz`

2. 解压并配置路径

使用tar命令解压

`tar -xzvf </path/to/kubernetes>`

并将解压后的文件移动到`/usr/local/`目录下

3. 配置环境变量

新增`/etc/profile.d/kubernetes.sh`文件，并写入

```
# add kubernetes binary to PATH
export PATH=/usr/local/kubernetes/node/bin:$PATH
```

运行`kubectl version`，打印kubernetes版本信息

## 配置kubernetes minion node

### 配置kubernetes基础信息

在`/etc/kubernetes/config`文件中写入：

```
# Comma separated list of nodes running etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://172.16.0.196:2379"
# Logging will be stored in system journal
KUBE_LOGTOSTDERR="--logtostderr=true"
# Journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"
# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"
# Api-server endpoint used in scheduler and controller-manager
KUBE_MASTER="--master=http://172.16.0.196:8080"
```

- etcd-servers: etcd的所在的ip及端口
- master: 指定master机器所在的ip及api server的端口，默认为8080

### 配置kube-proxy

1. 写入配置文件

在`/etc/kubernetes/proxy`文件中写入：

```
# Add your own!
KUBE_PROXY_ARGS=""
```

2. 配置成服务

将如下内容写入`/usr/lib/systemd/system/kube-proxy.service`

```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/kubernetes/node/bin/kube-proxy \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

3. 启动服务，并设置成开机自启动

```
systemctl start kube-proxy
systemctl enable kube-proxy
```

### 配置kubelet

1. 写入配置文件

在`/etc/kubernetes/kubelet`文件中写入：

```
# kubelet bind ip address(Provide private ip of minion)
KUBELET_ADDRESS="--address=0.0.0.0"
# port on which kubelet listen
KUBELET_PORT="--port=10250"
# leave this blank to use the hostname of server
KUBELET_HOSTNAME="--hostname-override=172.16.0.197"
# Location of the api-server
KUBELET_MASTER_CONFIG="--cgroup-driver=cgroupfs --kubeconfig=/etc/kubernetes/master-kubeconfig.yaml"
# pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry-vpc.cn-beijing.aliyuncs.com/rsq-main/pause-amd64:3.0"
# Add your own!
KUBELET_ARGS=""
```

- address: kubelet绑定的本地地址
- port: kubelet绑定的本地端口，需要与master中的配置一致
- hostname-override: 当前的node使用的hostname
- cgroup-driver: 注意，这里的driver需要与docker中的一致，使用systemd会报错，这里改成docker默认的cgroupfs
- kubeconfig: 指定一个yaml文件，来配置master的详细信息。1.6之后改用这种方式
- pod-infra-container-image: **注意**这里如果不明确指定，将会使用kubernetes官方默认的google镜像仓库，国内无法访问。因此这里可以改成国内的（比如阿里云的相应镜像）。具体修改方法可以参考[使用内部镜像](internal-image.md)

配置文件中的`kubeconfig`参数需要指定一个配置master的yaml文件，
新建`/etc/kubernetes/master-kubeconfig.yaml`文件如下：

```
 kind: Config
 clusters:
 - name: local
   cluster:
     server: http://172.16.0.196:8080
 users:
 - name: kubelet
 contexts:
 - context:
     cluster: local
     user: kubelet
   name: kubelet-context
 current-context: kubelet-context
```

2. 配置成服务

将如下内容写入`/usr/lib/systemd/system/kube-proxy.service`

```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/kubernetes/node/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_MASTER_CONFIG \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_POD_INFRA_CONTAINER \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

3. 启动服务，并设置成开机自启动

```
systemctl start kubelet
systemctl enable kubelet
```

## 启动minion node

```
for SERVICES in kube-proxy kubelet; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```