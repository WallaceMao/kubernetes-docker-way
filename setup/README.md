# 裸机安装kubernetes
kubernetes官方非常不推荐裸机直接安装kubernetes。不过折腾一遍，还是会对kubernetes的各个组件有比较深的了解。
**说明**
本文预设的环境是国内，无法访问google cloud的[镜像市场](http://gcr.io/)的情况

本安装说明具体分为以下过程：

- [前提条件](prerequisite.md)
- [安装kubernetes server](kubernetes-server.md)
- [安装kubernetes node](kubernetes-node.md)
- [安装flannel](flannel.md)
- [DNS配置](dns.md)
- [Dashboard](dashboard.md)

其他资料：

- [生成ca证书](gen-ca.md)
- [使用本地镜像](internal-image.md)