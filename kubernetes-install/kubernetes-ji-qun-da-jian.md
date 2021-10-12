# Kubernetes集群搭建

## 准备工作

### 环境准备

* OS：Ubuntu Server X64 20.04 LTS
* CPU：最低要求，1 CPU 2 核
* 内存：最低要求，4 GB
* 磁盘：最低要求，40 GB

### 关闭交换空间

```shell
swapoff -a
```

避免开机启动交换空间

```shell
# 注释 swap 开头的行
vi /etc/fstab
```

### 关闭防火墙

```shell
ufw disable
```

### 配置 DNS

```shell
# 取消 DNS 行注释，并增加 DNS 配置如：114.114.114.114，修改后重启下计算机
vi /etc/systemd/resolved.conf
```

### 安装 Docker

```shell
sudo apt-get update
sudo apt install docker.io
```

### 配置 Docker

通过修改 daemon 配置文件 `/etc/docker/daemon.json` 来使用加速器

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": [
    "https://lr5l2mfx.mirror.aliyuncs.com",
    "https://dockerhub.azk8s.cn"
  ],
  "storage-driver": "overlay2"
}
```

### 重启 Docker

```shell
systemctl daemon-reload
systemctl restart docker
```

### 安装 Kubernetes 必备工具

安装三个 Kubernetes 必备工具，分别为 **kubeadm**，**kubelet**，**kubectl**

```
# 安装系统工具
sudo apt-get update && apt-get install -y apt-transport-https ca-certificates curl

# 安装 GPG 证书
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add

# 写入软件源；注意：20.04系统代号为 Focal Fossa，但目前阿里云不支持，所以沿用 16.04 的 xenial
cat << EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

# 安装
apt-get update && apt-get install -y kubelet kubeadm kubectl
```

### 同步时间

* **设置时区**

```
dpkg-reconfigure tzdata
```

* 选择 **Asia（亚洲）**

![img](http://images.qfdmy.com/FsMRMpgYJ031zKrqU25EZZVUOXmG@.webp)

* 选择 **Shanghai（上海）**

![img](http://images.qfdmy.com/FmU9hGBfMj4Nni76\_qhS_MIBmpjY@.webp)

* #### **时间同步**

```
# 安装 ntpdate
apt-get install ntpdate

# 设置系统时间与网络时间同步（cn.pool.ntp.org 位于中国的公共 NTP 服务器）
ntpdate cn.pool.ntp.org

# 将系统时间写入硬件时间
hwclock --systohc
```

* #### **确认时间**

```
date

# 输出如下（自行对照与系统时间是否一致）
Sun Feb 23 12:05:17 CST 2020
```

### 修改 cloud.cfg

主要作用是防止重启后主机名还原

```
vi /etc/cloud/cloud.cfg

# 该配置默认为 false，修改为 true 即可
preserve_hostname: true
```

### 单独节点配置

> **注意：** 为 Master 和 Node 节点单独配置对应的 **IP** 和 **主机名**

### 配置 IP

编辑 `vi /etc/netplan/00-installer-config.yaml` 配置文件，修改内容如下

```
network:
    ethernets:
        ens33:
          addresses: [192.168.33.110/24]
          gateway4: 192.168.33.2
          nameservers:
            addresses: [192.168.33.2]
    version: 2
```

使用 `netplan apply` 命令让配置生效

### 配置主机名

```
# 修改主机名
hostnamectl set-hostname kubernetes-master

# 配置 hosts
cat >> /etc/hosts << EOF
192.168.81.110 kubernetes-master
EOF
```

## Kubernetes 安装集群

### 初始化master节点

```
kubeadm init --pod-network-cidr=172.172.0.0/16 --image-repository registry.aliyuncs.com/google_containers
```

### 异常

```bash
[ERROR ImagePull]: failed to pull image registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0: output: Error response from daemon: manifest for registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0 not found: manifest unknown: manifest unknown
```

```bash
docker pull coredns/coredns:1.8.0
docker tag coredns/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns:v1.8.0
```

配置 kubectl mkdir -p $HOME/.kube cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

### 非 ROOT 用户执行

chown $(id -u):$(id -g) $HOME/.kube/config

### 安装从节点

将 Node 节点加入到集群中很简单，只需要在 Node 服务器上安装 **kubeadm**，**kubectl**，**kubelet** 三个工具，然后使用 `kubeadm join` 命令加入即可

```
kubeadm join 192.168.81.110:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:3eac1be34c9e324279ebd843087e7dd002b3102c7d14313aec490cd73b4138ad
```

### 验证是否成功

回到 Master 主节点查看是否安装成功

> **注意：** 如果 Node 节点加入 Master 时配置有问题可以在 Node 节点上使用 `kubeadm reset` 重置配置再使用 `kubeadm join` 命令重新加入即可。希望在 master 节点删除 node ，可以使用 `kubeadm delete nodes <NAME>` 删除。

### 查看 Pods 状态

coredns 尚未运行，此时我们还需要安装网络插件

```
watch kubectl get pods -n kube-system -o wide
```

## Kubernetes 网络插件

### 安装Calico

### 下载 Calico 配置

```
wget https://docs.projectcalico.org/manifests/calico.yaml
```

### 安装网络插件 Calico

```
kubectl apply -f calico.yaml
```

### 验证安装是否成功

查看 Calico 网络插件处于 **Running** 状态即表示安装成功

```
watch kubectl get pods --all-namespaces
```

### 异常

```
coredns-545d6fc579-4fnw6 ImagePullBackOff
coredns-545d6fc579-x94qq ImagePullBackOff
```

node 中执行

```
docker pull coredns/coredns:1.8.0
docker tag coredns/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns:v1.8.0
```

### 异常

```
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused
```

**注释掉/etc/kubernetes/manifests下的kube-controller-manager.yaml和kube-scheduler.yaml的- – port=0**
