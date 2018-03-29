## 环境准备
### linux服务器裸机
学习起见，不需要对master做高可用，需要的linux服务器如下：
- master: ip: 172.16.0.196, OS: CentOS 7.4, 只作为master，不作为node
- minion1: ip: 172.16.0.197, OS: CentOS 7.4, 作为node
- minion2: ip: 172.16.0.198, OS: CentOS 7.4, 作为node

### SSH key
1. 为方便安装，需要master可以直接SSH访问minion机器，因此在master机器上使用
`ssh-keygen`命令创建密钥对，检查`~/.ssh`目录下是否生成`id_rsa`和`id_rsa.pub`文件
2. 将master上的`~/.ssh/id_rsa.pub`文件中的内容拷贝到minion机器的`~/.ssh/authorized_keys`
文件内

### 关闭swap
kubernetes1.8开始，要求关闭系统的swap。可以使用`free -m`检查。关闭的方式有以下两步：
- `swapoff -a`: 关闭当前系统运行环境
- 注释掉/etc/fstab文件中的swap行: 禁用启动时挂载swap