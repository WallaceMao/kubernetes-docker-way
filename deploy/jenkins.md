# jenkins集成

## 集成后的流程
1. 提交项目代码到github（其中包含Dockerfile）
2. jenkins拉去github代码，在服务器上构建docker镜像
3. 将构建好的docker镜像打tag，并上传到镜像仓库
4. jenkins通知kubernetes集群，更新镜像版本

## jenkins docker镜像操作说明

```
# build and push image
cd $WORK_DIR
RSQ_VERSION="${PROJECT_VERSION}.${BUILD_NUMBER}"
MAIN_IMAGE_TAG="${HUB_ROOT}${PROJECT_NAME}-main:${RSQ_VERSION}"

echo "WORK_DIR: ${WORK_IDR}"
echo "HUB_ROOT: ${HUB_ROOT}"
echo "PROJECT_NAME: ${PROJECT_NAME}"
echo "PROJECT_VERSION: ${PROJECT_VERSION}"
echo "RSQ_VERSION: ${RSQ_VERSION}"
echo "MAIN_IMAGE_TAG: ${MAIN_IMAGE_TAG}"

sudo docker build -f ./web/Dockerfile -t $MAIN_IMAGE_TAG ./web

# login and push to aliyun
sudo docker login --username=s-docker-image@rishiqing --password="${ALIYUN_DOCKER_PASSWORD}" registry-internal.cn-beijing.aliyuncs.com
sudo docker push $MAIN_IMAGE_TAG


# rolling update cluster
kubectl -n qywx set image deployment/qywx-isv-access-main-deploy qywx-isv-access-main=registry-vpc.cn-beijing.aliyuncs.com/rsq-project/${PROJECT_NAME}-main:${RSQ_VERSION}
```

说明：
- 在操作kubernetes执行rolling update时，使用`kubectl -n [namespace name] set image`的方式更新kubernetes容器的镜像