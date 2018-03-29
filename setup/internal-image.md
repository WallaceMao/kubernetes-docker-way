# 本地镜像问题
由于众所周知的原因，国内无法访问kubernetes默认的google镜像仓库，因此有两种选择：
- 使用本地镜像仓库
- 使用国内第三方镜像仓库

第一种方式可以自己搭建docker镜像仓库，在此不做说明。以下以阿里云镜像仓库为例，说明本地镜像的使用

## 镜像获取方式
国内的问题在于：国内不能直接访问google镜像市场。那么只能通过一个中介获取镜像，思路为：
- 找一个国内可以访问的镜像仓库（hubM）
- 找一个国内可以访问的、可以访问hubM，同时可以访问google镜像市场的服务器（ServerM)，并安装docker
- 在ServerM上讲google镜像市场的镜像pull到本地，然后打tag，push到hubM
- kubernetes服务器上直接从hubM拉去镜像

## 阿里云镜像中转方式

### 阿里云账号的创建

阿里云镜像仓库地址：[https://cr.console.aliyun.com/#/imageList](https://cr.console.aliyun.com/#/imageList)

### 本地pull并push到阿里云镜像仓库

```
docker pull kubernetes/pause
docker tag f9d5de079539 registry.cn-beijing.aliyuncs.com/rsq-main/pause-amd64:3.0
docker login --username=s-docker-image@rishiqing registry.cn-beijing.aliyuncs.com
docker push registry.cn-beijing.aliyuncs.com/rsq-main/pause-amd64:3.0
```

### kubernetes管理的node上，进行docker登录

```
docker login --username=s-docker-image@rishiqing registry-vpc.cn-beijing.aliyuncs.com
```

### 拉取成功后，设置pod镜像的基础仓库

修改kubelet的配置为如下：

```
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry-vpc.cn-beijing.aliyuncs.com/rsq-main/pause-amd64:3.0"
```

然后重启：

```
systemctl daemon-reload
systemctl restart kubelet
```

## google服务器镜像中转方式
国内是可以访问google服务器的，那么利用google服务器结合docker官方的hub，也可以做中转

### google服务器pull并push到docker hub镜像仓库

### kubernetes管理的node上，进行docker登录

### 拉取成功后，设置pod镜像的基础仓库