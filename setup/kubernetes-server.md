# master安装与配置
本章节中的安装过程只需要在master端操作

## 安装etcd
etcd是一款分布式的键值对存储工具。
kubernetes中etcd是核心组件，用来存储kubernetes object的信息。
在生产环境中，etcd需要做分布式，防止单点故障的产生。
安装过程参考：[etcd官方文档](https://github.com/coreos/etcd/releases/)

1. 下载安装etcd

```
    ETCD_VER=v3.3.2

    # choose either URL
    GOOGLE_URL=https://storage.googleapis.com/etcd
    GITHUB_URL=https://github.com/coreos/etcd/releases/download
    DOWNLOAD_URL=${GOOGLE_URL}

    rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
    rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

    curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
    tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
    rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

    /tmp/etcd-download-test/etcd --version

    ETCDCTL_API=3 /tmp/etcd-download-test/etcdctl version
```

## 安装kubernetes server
kubernetes的安装方式很多，本文下载官方发布的Binary版本的压缩包

1. 下载官方压缩包

在[kubernetes官方发布页面](https://kubernetes.io/docs/imported/release/notes/#server-binaries)找到`Server Binaries`压缩包的下载地址。
使用wget下载

`wget https://dl.k8s.io/v1.9.6/kubernetes-server-linux-amd64.tar.gz`

2. 解压并配置路径

使用tar命令解压

`tar -xzvf </path/to/kubernetes>`

并将解压后的文件移动到`/usr/local/`目录下

3. 配置环境变量

新增`/etc/profile.d/kubernetes.sh`文件，并写入

```
# add kubernetes path
export PATH=/usr/local/kubernetes/server/bin:$PATH
```

运行`kubectl version`，打印kubernetes版本信息

## 配置kubernetes master

### 配置etcd

1. 写入etcd配置文件

`vi /etc/etcd/etcd.conf`

写入以下配置信息：

```
#[member]
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

#[cluster]
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
```

2. 配置系统信息

执行以下命令，配置基础的etcd运行环境

```
mkdir /var/lib/etcd
mkdir /etc/etcd
groupadd -r etcd
useradd -r -g etcd -d /var/lib/etcd -s /sbin/nologin -c "etcd user" etcd
chown -R etcd:etcd /var/lib/etcd
```

3. 配置成服务

将`etcd`和`etcdctl`放入/usr/bin/下，并将如下内容写进`/usr/lib/systemd/system/etcd.service`文件

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

4. 启动etcd并校验，然后设置成开机启动

```
systemctl start etcd
systemctl status etcd
systemctl enable etcd
```

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

### 配置api server

1. 生成client ca证书和server ca证书
server ca证书用于tls链接；client ca证书用于客户端验证
具体的证书生成方式参照[gen-ca.md](gen-ca.md)

2. 写入配置文件

在`/etc/kubernetes/apiserver`文件中写入：

```
# Bind kube API server to this IP
KUBE_API_ADDRESS="--address=0.0.0.0"
# Port that kube api server listens to.
KUBE_API_PORT="--port=8080"
# Port kubelet listen on
KUBELET_PORT="--kubelet-port=10250"
# Address range to use for services(Work unit of Kubernetes)
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.0.0.0/16"
# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
# Add your own!
KUBE_API_ARGS="--client-ca-file=/srv/kubernetes/ca.crt --tls-cert-file=/srv/kubernetes/server.crt --tls-private-key-file=/srv/kubernetes/server.key"
```

- address: api server绑定的ip
- port: api server监听的端口
- service-cluster-ip-range: kubernetes的server的ip范围
- admission-control: 权限控制时的授权队列
- client-ca-file: 客户端验证的ca证书
- tls-cert-file: TLS验证的服务器端证书
- tls-private-key-file: TLS验证的私钥。

3. 配置成服务

将如下内容写入`/usr/lib/systemd/system/kube-apiserver.service`

```
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/kubernetes/server/bin/kube-apiserver \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_ETCD_SERVERS \
            $KUBE_API_ADDRESS \
            $KUBE_API_PORT \
            $KUBELET_PORT \
            $KUBE_ALLOW_PRIV \
            $KUBE_SERVICE_ADDRESSES \
            $KUBE_ADMISSION_CONTROL \
            $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

4. 启动服务，并设置成开机自启动

```
systemctl start kube-apiserver
systemctl enable kube-apiserver
```

### 配置controller manager

1. 写入配置文件

在`/etc/kubernetes/controller-manager`文件中写入：

```
# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--root-ca-file=/srv/kubernetes/ca.crt --service-account-private-key-file=/srv/kubernetes/server.key"
```

- root-ca-file: 客户端验证的ca证书
- service-account-private-key-file: 用来sign ServiceAccount token的密钥，如果不提供，那么默认就是server private key

2. 配置成服务

将如下内容写入`/usr/lib/systemd/system/kube-controller.service`

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/local/kubernetes/server/bin/kube-controller-manager \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

3. 启动服务，并设置成开机自启动

```
systemctl start kube-controller
systemctl enable kube-controller
```

### 配置scheduler

1. 写入配置文件

在`/etc/kubernetes/scheduler`文件中写入：

```
# Add your own!
KUBE_SCHEDULER_ARGS=""
```

2. 配置成服务

将如下内容写入`/usr/lib/systemd/system/kube-scheduler.service`

```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/kubernetes/server/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

3. 启动服务，并设置成开机自启动

```
systemctl start kube-scheduler
systemctl enable kube-scheduler
```

## 启动master

master中需要启动etcd/apiserver/controller-manager/scheduler

```
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```