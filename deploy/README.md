# 部署过程
本文重点说明如何将一个应用部署到阿里云的kubernetes集群中

## 应用拆分服务
一个web应用一般会由多个服务组成，例如数据库、mq队列、redis缓存、基础配置等，在做kubernetes集群部署的时候，需要将各个服务拆分到kubernetes相应的单元中。具体的拆分方式参照[这里](arch.md)

## 部署步骤
从项目源码部署到阿里云的kubernetes集群中，可以参考[以下步骤](steps.md)

## jenkins集成步骤
kubernetes可以方便地与jenkins进行集成，参考[这里](jenkins.md)