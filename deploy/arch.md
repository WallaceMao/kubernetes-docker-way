# 应用的服务结构划分

## 数据库
一般应用可以直接使用阿里云的RDS作为数据源，方便易维护，降低了很多数据库的运维成本。如果确实需要独立安装的数据库，可以在kubernetes中部署数据库镜像。例如[MySQL的镜像](https://hub.docker.com/_/mysql/)。之后需要将数据库通过Service的方式暴露出来供其他Pod调用。

## 密钥
对于用户名、密码等机密信息，可以使用k8s的Secret来存储。例如k8s集群中用来拉取docker镜像时，需要提供镜像仓库的密钥，这个密钥就是存储在Secret中。

## 配置文件
几乎任何应用都会有配置文件，里面可能会包含数据库连接地址、全局变量、页面跳转页地址、mq地址等。对于直接在服务器上部署的应用，一般配置文件会以文件的方式独立存储在服务器的某个路径下，应用启动会读取这个路径下的配置文件即可。而对于k8s集群，建议使用k8s的ConfigMap组件，来进行配置文件的统一存储，保证多个应用的配置是相同的。

## 消息队列
对于消息队列，可以直接使用mq相关的docker镜像，部署到k8s集群中。mq的端口可以通过Service的方式暴露出来，以供其他Pod调用。一般情况下，Service暴露的端口仅供Pod内网调用，不可外网访问。

## 缓存
对于redis等缓存，可以直接使用相关的docker镜像，部署到k8s集群中。其端口可以通过Service的方式暴露出来，以供其他Pod调用。一般情况下，Service暴露的端口仅供Pod内网调用，不可外网访问。

## 主应用后台
主应用后台以docker镜像的方式发布，以Service的方式将对外服务端口暴露出来。主应用的对外服务端口需要SLB或者nginx反向代理等暴露到公网。

## 前端资源
对于web的前端相关资源，推荐直接使用OSS存储，这样可以充分利用CDN的功能。对于一些无法放到OSS中的前端资源（例如html页面等），则需要使用下面的“nginx反向代理”来做转发。

## nginx反向代理
通常情况下，多个服务的应用需要使用nginx做转发。对于不需要外网访问的服务，可以直接使用Pod网络的DNS进行内网访问。而对于需要多个公网访问的端口，可以使用nginx镜像做转发。例如前端页面转发到前端Service，后台转发到后台Service，那么可以部署一个nginx服务，做统一转发。

## 其他