# flannel安装及配置
kubernetes在物理机器网络之上独立出来了一个扁平化的虚拟网络，这个网络实现了pod之间的通讯。
flannel模块实现了这样一个虚拟网络。
flannel在master和minion node上均需要安装

## 安装flannel

直接使用：

`yum install flannel`

安装，目前yum源上的最新版本为0.7.1

## 配置flannel

`/etc/sysconfig/flanneld`文件：

```
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://172.16.0.196:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
```

- FLANNEL_ETCD_ENDPOINTS: 指定了etcd的地址
- FLANNEL_ETCD_PREFIX：etcd中的路径

## 在etcd中添加网络信息

在etcd所在的主机上，运行：

```
etcdctl mkdir /kube-centos/network
etcdctl mk /kube-centos/network/config "{ \"Network\": \"172.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"
```

其中Network为划分的pod所在的子网的地址，即flannel网络都会位于172.30.xxx.xxx下

## 启动flannel

```
systemctl start flanneld
systemctl enable flanneld
```

启动后flannel会在node上创建相应的环境变量

## 配置docker网络

1. 启动flannel成功后，看一下`/run/flannel/subnet.env`，示例如下：

```
FLANNEL_NETWORK=172.30.0.0/16
FLANNEL_SUBNET=172.30.40.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
```

执行以下命令：

```
source /run/flannel/subnet.env
ifconfig docker0 $FLANNEL_SUBNET
```

docker安装后，会在node上安装docker0网卡。以上命令设置docker0网卡地址为flannel子网地址

2. 配置docker，保证每次启动docker的时候会加载flannel的配置

打开`/usr/lib/systemd/system/docker.service`，并增加一下环境变量文件：

```
EnvironmentFile=/run/flannel/subnet.env
```

同时修改`ExecStart`为：

```
ExecStart=/usr/bin/dockerd --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```

3. 重新加载配置文件，重新启动docker

```
systemctl daemon-reload
systemctl restart docker
```

## 测试是否成功

新建pod并查看pod的ip地址