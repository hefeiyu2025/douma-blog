---
tags:
  - ubuntu
  - docker
  - k8s
  - 教程
  - 技巧
title: Ubuntu安装docker&K8S
createTime: 2024/09/24 16:33:23
permalink: /article/pwgfndjk/
---

## Ubuntu安装docker&K8S

### 1. 安装最新版docker

```bash
#获取语句并执行
curl -fsSL "https://get.docker.com/" | sh
```

## 2. 安装指定版本docker

```bash
# 清理已安装版本
apt-get remove docker docker-engine docker-ce docker.io docker-ce docker-ce-cli
# 使 apt 可以通过 https 使用 repository
apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# 注意二选一
# 添加Docker官方的GPG密钥并更新索引包
curl -fsSL https://download.docker.com/linux/ubuntu/gpg |  apt-key add -
# 设置stable存储库
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"


# 阿里云镜像（上下二选一）
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg |  apt-key add -
# 设置stable存储库
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"


# 更新源
apt-get update
#列出docker可用版本
apt-cache madison docker-ce docker-ce-cli
# 替换<VERSION> ，安装指定版本，示例： apt-get -y install docker-ce=5:18.09.9~3-0~ubuntu-bionic docker-ce-cli=5:18.09.9~3-0~ubuntu-bionic containerd.io
apt-get -y install docker-ce=<VERSION> docker-ce-cli=<VERSION> containerd.io 
# 启动docker服务
systemctl start docker
# 检查安装成功
docker info
```

## 3. 安装指定版本kubelet kubeadm kubectl

```bash
# 使 apt 可以通过 https 使用 repository
apt-get install -y apt-transport-https ca-certificates curl software-properties-common
# 添加K8S阿里云镜像
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF 
# 更新源
apt-get update
#列出可用版本
apt-cache madison kubelet kubeadm kubectl
#安装最新版
apt-get  -y install kubelet kubeadm kubectl
# 安装指定版本：
apt-get  -y install kubeadm=<VERSION> kubectl=<VERSION> kubelet=<VERSION>
# 检查是否安装成功
kubeadm version 
kubectl version 
kubelet version
```

## 4. 安装NFS（ Network File System）网络文件系统

### 服务端

```bash
# 安装nfs-server(服务端) nfs-common（客户端）
apt-get install nfs-kernel-server nfs-common
# 修改共享挂载目录
vim /etc/exports
#在文件末位追加 &nfsdir& 这个需修改为共享的文件夹，并且需要先创建好，否则后面会启动失败;* 代表所有IP适用，若要修改为指定机器，直接填写IP地址
# /srv/homes     hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
&nfsdir&  * (rw,sync,no_root_squash)
#重启服务 （若遇到启动失败，请先检查目录是否已经创建）
/etc/init.d/nfs-kernel-server stop
/etc/init.d/rpcbind stop
/etc/init.d/rpcbind start
/etc/init.d/nfs-kernel-server start
```

### 客户端

```bash
#安装nfs-common（客户端）
apt-get install nfs-common
# 查看NFS服务器共享目录 &nfs-server-ip& 服务端IP地址
showmount -e &nfs-server-ip&
# 挂载目录&nfs-server-ip& 服务端IP地址  &nfsdir& 服务端共享的文件夹 &localdir& 挂载本地的文件夹
mount &nfs-server-ip&:&nfsdir&   &localdir&
#开机自动挂载
vim /etc/fstab
# 将下列语句追加到文件末 &nfs-server-ip& 服务端IP地址  &nfsdir& 服务端共享的文件夹 &localdir& 挂载本地的文件夹
&nfs-server-ip&:&nfsdir&   &localdir&      nfs    rw    0     0

#重启服务 （若遇到启动失败，请先检查目录是否已经创建）
/etc/init.d/rpcbind stop
/etc/init.d/rpcbind start
```

## 5.安装K8S集群管理

参考网址：https://kuboard.cn

前提条件：

1. 安装好docker环境
2. 安装kubelet kubeadm kubectl
3. 安装nfs
4. 关闭防火墙，关闭交换分区

#### 1.修改sysctl.conf

```
# 修改 /etc/sysctl.conf
# 如果有配置，则修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
# 可能没有，追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
# 执行命令以应用
sysctl -p
```

#### 2.安装master1

```
# 替换 x.x.x.x 为 ApiServer LoadBalancer 的 IP 地址
export APISERVER_IP=x.x.x.x
# 替换 apiserver.demo 为 前面已经使用的 dnsName
export APISERVER_NAME=apiserver.k8s
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/16
echo "${APISERVER_IP}  ${APISERVER_NAME}" >> /etc/hosts


# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.1
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "${APISERVER_NAME}:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs
```

执行结果

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join apiserver.k8s:6443 --token xfz9vl.6mjv1hgdtpl6vl5b \
    --discovery-token-ca-cert-hash sha256:760e4d2c6a3a4db6ef0f338af9feb57281b21af58eb9fde03ccfb03d4ddf0c6e \
    --control-plane --certificate-key 91e3f48a87b70e074986aa6cbb6468005fd20e026b4d89331b7fff56a7b28f25

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join apiserver.k8s:6443 --token xfz9vl.6mjv1hgdtpl6vl5b \
    --discovery-token-ca-cert-hash sha256:760e4d2c6a3a4db6ef0f338af9feb57281b21af58eb9fde03ccfb03d4ddf0c6e 
```

复制配置文件

```
# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config
```

安装网络插件

```bash
# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/v3.8/getting-started/kubernetes/
rm -f calico.yaml
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
sed -i "s#192\.168\.0\.0/16#${POD_SUBNET}#" calico.yaml
kubectl apply -f calico.yaml
```

检查master 初始化结果

```bash
# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes
```

#### 3.安装master2

```bash
# 替换 x.x.x.x 为 ApiServer LoadBalancer 的 IP 地址 master1 的地址
export APISERVER_IP=x.x.x.x
# 替换 apiserver.k8s 为 前面已经使用的 dnsName
export APISERVER_NAME=apiserver.k8s
echo "${APISERVER_IP}    ${APISERVER_NAME}" >> /etc/hosts
# 使用前面步骤中获得的master2 节点的 join 命令，在master1 init 的执行结果中查找
kubeadm join apiserver.k8s:6443 --token xfz9vl.6mjv1hgdtpl6vl5b \
    --discovery-token-ca-cert-hash sha256:760e4d2c6a3a4db6ef0f338af9feb57281b21af58eb9fde03ccfb03d4ddf0c6e \
    --control-plane --certificate-key 91e3f48a87b70e074986aa6cbb6468005fd20e026b4d89331b7fff56a7b28f25
```

#### 4.安装worker1,worker2

```bash
# 替换 x.x.x.x 为 ApiServer LoadBalancer 的 IP 地址 master1 的地址
export APISERVER_IP=x.x.x.x
# 替换 apiserver.k8s 为 前面已经使用的 dnsName
export APISERVER_NAME=apiserver.k8s
echo "${APISERVER_IP}    ${APISERVER_NAME}" >> /etc/hosts
# 使用前面步骤中获得worker 节点的 join 命令，在master1 init 的执行结果中查找
kubeadm join apiserver.k8s:6443 --token xfz9vl.6mjv1hgdtpl6vl5b \
    --discovery-token-ca-cert-hash sha256:760e4d2c6a3a4db6ef0f338af9feb57281b21af58eb9fde03ccfb03d4ddf0c6e 
```

#### 5.移除worker 节点

在准备移除的 worker 节点上执行

```bash
kubeadm reset
```

在master 节点 上执行

```bash
# 查看 master 节点worker节点名称
kubectl get nodes
kubectl delete node $node-name$
```

#### 6.安装Kuboard

安装

```bash
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
```

卸载

```bash
kubectl delete -f https://kuboard.cn/install-script/kuboard.yaml
```

kuboard.yaml 文件内容

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuboard
  namespace: kube-system
  annotations:
    k8s.eip.work/displayName: kuboard
    k8s.eip.work/ingress: "true"
    k8s.eip.work/service: NodePort
    k8s.eip.work/workload: kuboard
  labels:
    k8s.eip.work/layer: monitor
    k8s.eip.work/name: kuboard
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s.eip.work/layer: monitor
      k8s.eip.work/name: kuboard
  template:
    metadata:
      labels:
        k8s.eip.work/layer: monitor
        k8s.eip.work/name: kuboard
    spec:
      containers:
      - name: kuboard
        image: eipwork/kuboard:latest
        imagePullPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: kuboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 32567
  selector:
    k8s.eip.work/layer: monitor
    k8s.eip.work/name: kuboard

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kuboard-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kuboard-user
  namespace: kube-system

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kuboard-viewer
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: kuboard-viewer
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-viewer-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- kind: ServiceAccount
  name: kuboard-viewer
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-viewer-pvp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:persistent-volume-provisioner
subjects:
- kind: ServiceAccount
  name: kuboard-viewer
  namespace: kube-system

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuboard
  namespace: kube-system
  annotations:
    nginx.org/websocket-services: "kuboard"
    nginx.com/sticky-cookie-services: "serviceName=kuboard srv_id expires=1h path=/"
spec:
  rules:
  - host: kuboard.yourdomain.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuboard
          servicePort: http
```

获取Token

1. 管理员

```bash
   #此Token拥有 ClusterAdmin 的权限，可以执行所有操作
   kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}')   
   # 取输出信息中 token 字段
```

2. 只读用户

```bash
   #拥有的权限
   #	view 可查看名称空间的内容
   #	system:node 可查看节点信息
   #	system:persistent-volume-provisioner 可查看存储类和存储卷声明的信息
   kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kuboard-viewer | awk '{print $1}')   
   # 取输出信息中 token 字段
```

需要等待分钟，待容器安装完成，安装完成后访问http://{节点IP}/:32567
