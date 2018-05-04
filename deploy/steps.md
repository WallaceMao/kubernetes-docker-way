# 阿里云kubernetes部署步骤
关于kubernetes的操作步骤，有三种：命令行、命令对象配置和声明对象配置。具体说明参照[官方说明](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/#imperative-object-configuration)。命令行的方式由于难以追踪，已经被强烈不建议在生产环境中使用。为方便起见，本文统一采用命令对象配置（imperative object configuration）

## 本地构建镜像
1. 在源码项目中增加Dockerfile  
示例：  
Dockerfile:  
```
FROM registry-internal.cn-beijing.aliyuncs.com/rsq-public/tomcat:8.0.50-jre8

LABEL name="qywx-isv-access-main" \
       description="backend for integration of rishiqing and qywx(enterprise version of WeChat)" \
       maintainer="Wallace Mao" \
       org="rishiqing"

# set default time zone to +0800 (Shanghai)
ENV TIME_ZONE=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TIME_ZONE /etc/localtime && echo $TIME_ZONE > /etc/timezone

ENV CATALINA_HOME /usr/local/tomcat
WORKDIR $CATALINA_HOME

ADD target/qywxbackauth-main-*.war webapps/qywxbackauth.war

CMD ["catalina.sh", "run"]

```

**注意：一般的docker镜像中的默认时区为0时区，为了避免不必要的麻烦，一般在中国可以设置成北京时间（Asia/Shanghai）**

2. 在本地构建镜像，测试。

```
sudo docker build -f [Dockerfile path] -t [image name]:[image tag] [buid directory]
```

其中：
- Dockerfile path：Dockerfile所在的路径，一般是相对路径
- image name：构建的镜像的名称。由于上传镜像仓库时根据名称上传到相应镜像仓库，所以这里的name需要包含镜像仓库的地址。例如：registry-internal.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-main。注意对于阿里云，这里的镜像仓库的地址建议使用阿里云镜像仓库的内网地址。详细说明参照“镜像仓库操作”
- image tag：镜像的标签，强烈不建议使用lastest的默认标签，而应该指定版本号。例如：0.1.0.29.32。如果集成jenkins的话，可以将jenkins的BUILD_NUMBER写到版本号中，以此来保证每次构建版本号都会更新
- build directory：构建的目录，如果是当前目录，一般指定“.”即可

构建的示例命令：

```
sudo docker build -f ./Dockerfile -t registry-internal.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-main:0.1.0.29.32 .
```

## 镜像仓库操作
1. 登录阿里云控制台，找到镜像仓库页面。地址为：[https://cr.console.aliyun.com/#/imageList](https://cr.console.aliyun.com/#/imageList)。选择“华北2”区，选择相应的命名空间。目前的命名空间有：

- rsq-main：日事清主仓库，一般用存储日事清应用可以源码的镜像
- rsq-project：日事清主仓库，一般用来存储私有镜像
- rsq-public：第三方依赖的镜像，为了加快阿里云拉去镜像的速度，可以放到这个命名空间下做缓存

2. 在指定命名空间下新建仓库，仓库名称需要与项目名称一致。代码源可以选择本地仓库。

3. 本地使用docker登录镜像仓库。  
例如：username使用实际的用户名，并输入密码  
```
sudo docker login --username=u-wallace@rishiqing registry.cn-beijing.aliyuncs.com
```

4. 本地将构建好的镜像推送到镜像仓库。  
例如：
```
sudo docker push registry.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-main:[镜像版本号]
```

5. 在镜像仓库中检查，镜像是否已经推送成功

## kubernetes操作
**注意：由于本文采用的是命令对象的方式来创建kubernetes对象，所以一定要注意保留好相关的yaml说明文件，便于后续查找问题，更新恢复。**

1. 登录kubernetes集群

2. 在kubernetes集群中创建namespace

```
kubectl get namespaces
```

查看当前所有的集群命名空间。如果需要，可以通过以下命令新建namespace

```
kubectl create namespace [name space name]
```

3. 创建配置的ConfigMap  
新建ConfigMap的说明文件，示例如下：  
config.yaml：
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: [config-name]
data:
  config.properties: |
    jdbc.username=user
    jdbc.password=xxxxxxxx
    jdbc.url=jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf8&useSSL=true

    isv.app.id=1
    isv.rsq.url.root=https://www.rishiqing.com
```

**注意：metadata中name要改成自定义的name。data中，config.properties为实际注入到docker容器中的配置文件的名称，需要根据根据实际需要做修改**

使用kubectl命令创建ConfigMap
```
kubectl -n [namespace name] create -f config.yaml
```

**注意：一定要加“-n”参数来指定命名空间，否则新建的ConfigMap将会默认在default的命名空间中！**

检查命名空间中是否有ConfigMap
```
kubectl -n [namespace name] get configmaps
```

4. 通过docker镜像创建各Deployment  
根据之前拆分的服务，有几个需要启动的docker镜像，就需要新建几个Deployment。Deployment会自动维护Pod。典型的Deployment的说明文件如下：  
qywx-isv-access-main-deploy.yaml文件：  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qywx-isv-access-main-deploy
  labels:
    app: qywx-isv-access-main
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qywx-isv-access-main
  template:
    metadata:
      labels:
        app: qywx-isv-access-main
    spec:
      containers:
      - name: qywx-isv-access-main
        image: registry-vpc.cn-beijing.aliyuncs.com/rsq-project/qywx-isv-access-main:0.1.0.26.29
        volumeMounts:
        - name: base-config
          mountPath: /root/qywx
        ports:
        - containerPort: 8080
        restartPolicy: always
        livenessProbe:
          httpGet:
            path: /qywxbackauth/checkpreload.html
            port: 8080
          initialDelaySeconds: 90
          timeoutSeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /qywxbackauth/checkpreload.html
            port: 8080
          initialDelaySeconds: 30
          timeoutSeconds: 1
          periodSeconds: 5
        resources:
          requests:
            memory: "1024Mi"
            cpu: "500m"
          limits:
            memory: "1024Mi"
            cpu: "500m"
      volumes:
      - name: base-config
        configMap:
          name: qywx-main-config
      imagePullSecrets:
      - name: ali-docker
```

说明：
- metadata.labels：Deployment的label，名称要慎重，不要跟已有的deployment重复
- spec.selector.matchLabels：该Deployment所匹配的Pod的label，与spec.template.metadata.labels中的一定要匹配上。
- spec.template.spec.volumes中为注入到容器中的文件。在spec.template.spec.containers[0].volumeMounts中引用这个volumes。以上示例中，容器中将qywx-main-config这个ConfigMap注入到`/root/qywx`这个路径下。
- 强烈建议配置livenessProbe和readiness这两个配置项。livenessProbe决定了Pod什么时候会被重启，当httpGet返回失败的时候，Pod将会被重启；readinessProbe决定了启动Pod时，何时会被判断为Pod ready并提供对外服务。
- spec.template.spec.containers[0].resources：用来限制容器可使用的资源
- spec.template.spec.imagePullSecrets用来选择拉去docker镜像的密钥

使用kubectl命令创建Deployment
```
kubectl -n [namespace name] create -f qywx-isv-access-main-deploy.yaml
```

**注意：一定要加“-n”参数来指定命名空间，否则新建的Deployment将会默认在default的命名空间中！**

检查命名空间中是否有Deployment

```
kubectl -n [namespace name] get deployments
```

5. 需要对外暴露的服务，创建Service。  
需要对外暴露的服务，通过Service暴露端口，Service的声明文件示例如下：

service-qywx-isv-access-main.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: qywx-isv-access-main
  labels:
    app: qywx-isv-access-main
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30002
  selector:
    app: qywx-isv-access-main

```

注意：
- spec.type：鉴于阿里云的kubernetes负载均衡兼容性不好，建议目前选择NodePort
- spec.ports.port：对于集群内网访问该服务使用的端口
- spec.ports.targetPort：提供服务的容器所使用的端口
- spec.ports.nodePort：node主机上映射的端口，可以通过SLB将外网请求转发到该端口上，实现服务的外网访问

## 开放对外端口
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