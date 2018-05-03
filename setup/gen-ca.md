# 证书生成
如果不是用证书，仍然可以启动kubernetes，但是很多权限相关的功能将会受限，因此推荐在搭建kubernetes时生成证书。
但是生成证书并不是一件容易的事情，以下步骤是使用openssl工具进行证书生成的步骤。
强烈推荐在证书生成前，先看TLS原理

## 生成步骤
1. 生成ca根证书私钥ca.key

`openssl genrsa -out ca.key 2048`

2. 生成ca根证书ca.crt

`openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt`

- MASTER_IP是master的ip，例如172.16.0.196
- days是有效的天数

3. 生成server.key

`openssl genrsa -out server.key 2048`

4. 创建server csr配置文件

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = <country>
ST = <state>
L = <city>
O = <organization>
OU = <organization unit>
CN = <MASTER_IP>

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = <MASTER_IP>
IP.2 = <MASTER_CLUSTER_IP>

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

配置文件的参考示例为：

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = CN
ST = BJ
L = BJ
O = Rishiqing
OU = IT
CN = 172.16.0.196

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = 172.16.0.196
IP.2 = 10.0.0.1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

其中：
（1）<MASTER_IP>为master服务器的ip，例如：172.16.0.196
（2）<MASTER_CLUSTER_IP>为`--service-cluster-ip-range`参数指定的service CIDR中的第一个地址，例如：10.0.0.1

5. 根据server csr配置文件生成server.csr

`openssl req -new -key server.key -out server.csr -config csr.conf`

6. 生成server.crt

`openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000 -extensions v3_ext -extfile csr.conf`

7. 验证server.crt

`openssl x509  -noout -text -in ./server.crt`

8. 将ca.key/ca.crt/server.key/server.crt拷贝到相关目录

## 说明
- ca.crt主要用来做客户端验证，因此可以直接根据master ip生成根证书
- server.crt用来做TLS链接，本例中使用的是自签名。如果条件允许可以申请权威的https认证机构进行签名