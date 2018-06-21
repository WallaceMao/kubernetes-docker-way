# 阿里云docker容器集群部署说明（前端）

阿里云docker容器集群的部署具体分为两步：
1. docker镜像的构建与上传
2. kubernetes服务的创建与更新

## 1 docker镜像的构建与上传
### 1.1 在源码项目中增加Dockerfile  

将前端的html/js/css等文件打包到一个docker镜像中，并将该镜像上传到docker仓库。本文中使用的是阿里云的镜像仓库。  

示例：  
Dockerfile:  
```
FROM registry-internal.cn-beijing.aliyuncs.com/rsq-public/nginx:1.13.12

LABEL name="qywx-isv-access-nginx" \
       description="frontend server for integration of rishiqing and qywx(enterprise version of WeChat)" \
       maintainer="Wallace Mao" \
       org="rishiqing"

# set default time zone to +0800 (Shanghai)
ENV TIME_ZONE=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TIME_ZONE /etc/localtime && echo $TIME_ZONE > /etc/timezone

ENV HTML_ROOT /usr/share/nginx/html
ENV CONFIG_ROOT /etc/nginx/conf.d
WORKDIR $HTML_ROOT

# use template file as default config file
# COPY config/nginx.template $CONFIG_ROOT/default.conf
COPY dist .
```

**注意：**  
- **示例中FROM使用的是内网地址，需要改成公网地址**
- **LABEL中需要简要说明镜像细节和负责人，便于以后出现问题可以找到相关维护人员**
- **一般的docker镜像中的默认时区为0时区，为了避免不必要的麻烦，一般在中国可以设置成北京时间（Asia/Shanghai）**

### 1.2 在本地构建镜像，测试。

```
sudo docker build -f [Dockerfile path] -t [image name]:[image tag] [buid directory]
```

其中：
- Dockerfile path：Dockerfile所在的路径，一般是相对路径
- image name：构建的镜像的名称。由于上传镜像仓库时根据名称上传到相应镜像仓库，所以这里的name需要包含镜像仓库的地址。例如：registry-internal.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-nginx。注意对于阿里云，这里的镜像仓库的地址建议使用阿里云镜像仓库的内网地址。详细说明参照“镜像仓库操作”
- image tag：镜像的标签，强烈不建议使用lastest的默认标签，而应该指定版本号。例如：0.1.0.29.32。如果集成jenkins的话，可以将jenkins的BUILD_NUMBER写到版本号中，以此来保证每次构建版本号都会更新
- build directory：构建的目录，如果是当前目录，一般指定“.”即可

构建的示例命令：

```
sudo docker build -f ./Dockerfile -t registry-internal.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-nginx:0.1.0.29.32 .
```

### 1.3 登录并新建镜像仓库

1. 登录阿里云控制台，找到镜像仓库页面。地址为：[https://cr.console.aliyun.com/#/imageList](https://cr.console.aliyun.com/#/imageList)。选择“华北2”区，选择相应的命名空间。目前的命名空间有：

- rsq-main：日事清主仓库，一般用存储日事清应用可以源码的镜像
- rsq-project：日事清主仓库，一般用来存储私有镜像
- rsq-public：第三方依赖的镜像，为了加快阿里云拉去镜像的速度，可以放到这个命名空间下做缓存

2. 在指定命名空间下新建仓库，仓库名称需要与项目名称一致。代码源可以选择本地仓库。

### 1.4 推送本地镜像至镜像仓库

1. 本地使用docker登录镜像仓库。  
例如：username使用实际的用户名，并输入密码  
```
sudo docker login --username=s-docker-image@rishiqing registry.cn-beijing.aliyuncs.com
```

2. 本地将构建好的镜像推送到镜像仓库。  
例如：
```
sudo docker push registry.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-nginx:[镜像版本号]
```

3. 在镜像仓库中检查，镜像是否已经推送成功

## 2 kubernetes服务的创建与更新

kubernetes集群基本概念：  
- 集群。kubernetes集群，由一组物理服务器组成。
- 命名空间。在同一个集群内，根据功能划分成的不同区间。命名空间由kubernetes自动进行管理。

阿里云kubernetes集群的部署主要有两种方式，CMD（命令行）的方式和GUI（图形化界面）的方式。  

### 2.1 CMD方式

#### 2.1.1 登录kubernetes集群

#### 2.1.2 查看或创建命名空间

```
kubectl get namespaces
```

查看当前所有的集群命名空间。如果需要，可以通过以下命令新建namespace

```
kubectl create namespace [name space name]
```

#### 2.1.3 通过docker镜像创建Deployment  

编写deployment描述文件，模板如下：
qywx-isv-access-nginx-deploy.yaml文件：  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qywx-isv-access-nginx-deploy
  labels:
    app: qywx-isv-access-nginx
spec: 
  replicas: 2
  selector:
    matchLabels:
      app: qywx-isv-access-nginx
  template:
    metadata:
      labels:
        app: qywx-isv-access-nginx
    spec:
      restartPolicy: Always
      containers:
      - name: qywx-isv-access-nginx
        image: registry-vpc.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-nginx:0.0.1
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 60
          timeoutSeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          timeoutSeconds: 1
          periodSeconds: 5
#        resources:
#          requests:
#            memory: "1024Mi"
#            cpu: "500m"
#          limits:
#            memory: "1024Mi"
#            cpu: "500m"
      nodeSelector:
        workbei.service.rishiqing.com: workbei
      imagePullSecrets:
      - name: ali-docker
```

说明：
- metadata.labels：Deployment的label，名称要慎重，不要跟已有的deployment重复
- spec.selector.matchLabels：该Deployment所匹配的Pod的label，与spec.template.metadata.labels中的一定要匹配上。
- 强烈建议配置livenessProbe和readiness这两个配置项。livenessProbe决定了Pod什么时候会被重启，当httpGet返回失败的时候，Pod将会被重启；readinessProbe决定了启动Pod时，何时会被判断为Pod ready并提供对外服务。
- spec.template.spec.containers[0].resources：用来限制容器可使用的资源
- spec.template.spec.imagePullSecrets用来选择拉去docker镜像的密钥

使用kubectl命令创建Deployment
```
kubectl -n [namespace name] create -f qywx-isv-access-nginx-deploy.yaml
```

**注意：一定要加“-n”参数来指定命名空间，否则新建的Deployment将会默认在default的命名空间中！**

检查命名空间中是否有Deployment

```
kubectl -n [namespace name] get deployments
```

#### 2.1.4 需要对外暴露的服务，创建Service。  
需要对外暴露的服务，通过Service暴露端口，Service的声明文件示例如下：

service-qywx-isv-access-nginx.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: qywx-isv-access-nginx
  labels:
    app: qywx-isv-access-nginx
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30106
  selector:
    app: qywx-isv-access-nginx
```

注意：
- spec.type：鉴于阿里云的kubernetes负载均衡兼容性不好，建议目前选择NodePort
- spec.ports.port：对于集群内网访问该服务使用的端口
- spec.ports.targetPort：提供服务的容器所使用的端口
- spec.ports.nodePort：node主机上映射的端口，可以通过SLB将外网请求转发到该端口上，实现服务的外网访问

#### 2.1.5 开放对外端口
由于目前的kubernetes Service使用的是NodePort的方式，因此通过阿里云负载均衡（SLB）将外网请求转发到Service的服务端口（nodePort），实现外网访问。

1. 新增阿里云负载均衡（SLB）实例  

2. 新增虚拟服务器组  
每个需要对外访问的服务构成一个服务器组。将kubernetes的slave节点选中作为服务器组的服务器，端口设置为kubernetes中Service声明文件中配置的spec.ports.nodePort的端口号

3. 在实例上添加监听  
注意：
- 前端协议选择http或者https。后端协议选择http。建议https的证书验证放到负载均衡的监听中处理。

4. 添加转发策略  
在监听上添加转发策略，可以根据域名和URL将请求转发到不同的虚拟服务器组中

5. 测试是否可以实现外网访问  

### 2.2 GUI方式

参照“容器服务” -> “Kubernetes” 中的相关操作。本文不做详细说明
