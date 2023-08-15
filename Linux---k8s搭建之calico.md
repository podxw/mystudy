# Linux高级---k8s搭建之calico

[TOC]

#### 一、配置ip、修改主机名

```
ip最好使用静态ip，用nat模式或桥接都行
```

#### 二、修改/etc/hosts文件

```shell
192.168.10.144 master
192.168.10.154 node1
192.168.10.155 node2
```

#### 三、关闭防火墙、selinux

```shell
service firewalld disable
sed -i '/^SELINUX=/ s/enforcing/disabled/'  /etc/selinux/config
reboot
```

#### 四、三台都互相配置免密登录

```shell
ssh-keygen
ssh-copy-id master
ssh-copy-id node1
ssh-copy-id node2
```

#### 五、关闭交换分区

k8s默认不允许使用交换分区(因为性能低)，否则k8s初始化失败，可以在安装时指定--ignore-preflight-errors=Swap来使用交换分区

```shell
[root@master ~]# swapoff -a
[root@master ~]# vim /etc/fstab		# 注释下面这行
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

#### 六、修改内核参数

```shell
[root@master ~]# modprobe br_netfilter
[root@master ~]# echo "modprobe br_netfilter" >> /etc/profile
[root@master ~]# cat > /etc/sysctl.d/k8s.conf <<EOF
 net.bridge.bridge-nf-call-ip6tables = 1
 net.bridge.bridge-nf-call-iptables = 1
 net.ipv4.ip_forward = 1
 EOF
[root@master ~]# sysctl -p /etc/sysctl.d/k8s.conf
```

配置的原因：

```shell
问题1：sysctl是做什么的？
在运行时配置内核参数
  -p   从指定的文件加载系统参数，如不指定即从/etc/sysctl.conf中加载

问题2：为什么要执行modprobe br_netfilter？
修改/etc/sysctl.d/k8s.conf文件，增加如下三行参数：
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

sysctl -p /etc/sysctl.d/k8s.conf出现报错：

sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory

解决方法：
modprobe br_netfilter

问题3：为什么开启net.bridge.bridge-nf-call-iptables内核参数？
在centos下安装docker，执行docker info出现如下警告：
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled

解决办法：
vim  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

问题4：为什么要开启net.ipv4.ip_forward = 1参数？
kubeadm初始化k8s如果报错：
"proc/sys/net/ipv4/ip_forword contents are not set to 1"
就表示没有开启ip_forward，需要开启。

net.ipv4.ip_forward是数据包转发：
出于安全考虑，Linux系统默认是禁止数据包转发的。所谓转发即当主机拥有多于一块的网卡时，其中一块收到数据包，根据数据包的目的ip地址将数据包发往本机另一块网卡，该网卡根据路由表继续发送数据包。这通常是路由器所要实现的功能。
要让Linux系统具有路由转发功能，需要配置一个Linux的内核参数net.ipv4.ip_forward。这个参数指定了Linux系统当前对路由转发功能的支持情况；其值为0时表示禁止进行IP转发；如果是1,则说明IP转发功能已经打开。
```

#### 七、配置阿里云docker的repo源

```shell
[root@master ~]# yum install -y yum-utils
[root@master ~]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
[root@master ~]# yum install -y yum-utils device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib-devel python-devel epel-release openssh-server socat  ipvsadm conntrack ntpdate telnet ipvsadm
```

#### 八、配置安装k8s组件需要的阿里云的repo源

```shell
[root@master ~]# vim  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
```

#### 九、配置时间同步

```shell
[root@master ~]# yum install ntpdate -y
# 跟网络时间做同步
[root@master ~]# ntpdate cn.pool.ntp.org
# 添加到计划任务
[root@master ~]# crontab -e		# 添加下面这行
* */1 * * * /usr/sbin/ntpdate   cn.pool.ntp.org
# 重启crond服务
[root@master ~]# service crond restart
```

# 安装docker服务

#### 一、安装docker-ce

```shell
yum install docker-ce-20.10.6 -y	# 我安装的是23.0.3
# 设置docker开机自启
systemctl start docker && systemctl enable docker.service
```

#### 二、配置docker镜像加速器和驱动

```shell
[root@master ~]# vim  /etc/docker/daemon.json
{
 "registry-mirrors":["https://rsbud4vc.mirror.aliyuncs.com","https://registry.dockercn.com","https://docker.mirrors.ustc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
} 
# 以上将docker文件驱动修改为systemd，默认为cgroupfs，kubelet默认使用systemd，两者必须一致才可以。
# 以下是原文件内容，作为备份
{
"registry-mirrors": ["https://registry.docker-cn.com"],
"insecure-registries" : ["47.120.37.21:8088"]
}
# 重启docker服务
[root@master ~]# systemctl daemon-reload  && systemctl restart docker
[root@master ~]# systemctl status docker
```

#### 三、安装初始化k8s需要的软件包

kubadm的版本决定k8s镜像的版本

```shell
yum install -y kubelet-1.20.6 kubeadm-1.20.6 kubectl-1.20.6
# 我安装的是以下版本
[root@master ~]# yum install -y kubelet-1.23.3 kubeadm-1.23.3 kubectl-1.23.3
# 设置kublet开机自启
[root@master ~]# systemctl enable kubelet
```

```shell
# 每个软件包的作用
# Kubeadm：kubeadm是一个工具，用来初始化k8s集群的
# kubelet：安装在集群所有节点上，用于启动Pod的
# kubectl：通过kubectl可以部署和管理应用，查看各种资源，创建、删除和更新各种组件
```

#### 四、kubeadm初始化k8s集群（只在master上操作）

```shell
# 可以把初始化k8s集群需要的离线镜像包上传到三台机器上，手动解压：
docker load -i k8simage-1-20-6.tar.gz
# 没有的也可以正常进行下面的操作
```

根据我们自己的需求修改配置：

```shell
[root@master ~]# kubeadm config print init-defaults > kubeadm.yaml
[root@master ~]# vim kubeadm.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.40.180 #控制节点的ip
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: xianchaomaster1 #控制节点主机名
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers #修改镜像源
kind: ClusterConfiguration
kubernetesVersion: v1.20.6
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16 #指定pod网段， 需要新增加这个
scheduler: {}
#追加如下几行
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

> **--image-repository registry.aliyuncs.com/google_containers**为保证拉取镜像不到国外站点拉取，手动指定仓库地址为registry.aliyuncs.com/google_containers。kubeadm默认从k8s.gcr.io拉取镜像。**如果本地有导入到的离线镜像，会优先使用本地的镜像。**
>
> **mode: ipvs** 表示kube-proxy代理模式是ipvs，如果不指定ipvs，会默认使用iptables，但是iptables效率低，所以我们生产环境建议开启ipvs，阿里云和华为云托管的K8s，也提供ipvs模式。**iptables成熟稳定，但性能一般，适合少量节点的集群；ipv4为高性能，适合多节点，对负载均衡有高要求的场景**

初始化k8s：

```shell
[root@master ~]# kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification
[init] Using Kubernetes version: v1.23.0
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 23.0.3. Latest validated version: 20.10
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

……(此处省略大段输出)

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.144:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:fa5923604b72b39224b33e0cf9ed373224a6cb192123d8a643b07b7019aef6a5 
```

根据输出提示执行：

```
[root@master ~]#   mkdir -p $HOME/.kube
[root@master ~]#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master ~]#   sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 五、将node加入到集群

根据输出提示的最后两行在两台node上执行：

```shell
kubeadm join 192.168.10.144:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:fa5923604b72b39224b33e0cf9ed373224a6cb192123d8a643b07b7019aef6a5
# --ignore-preflight-errors=SystemVerification这个参数可能加一下比较好
# 如果忘了可以使用下面的这行命令重新生成
kubeadm token create --print-join-command
```

在master上执行：

```shell
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES                  AGE     VERSION
master   NotReady   control-plane,master   4m28s   v1.23.3
node1    NotReady   <none>                 29s     v1.23.3
node2    NotReady   <none>                 29s     v1.23.3
```

- 此时集群状态还是NotReady状态，因为没有安装网络插件。

- 可以看到node节点的ROLES角色为空，就表示这个节点是工作节点。

在master上通过以下命令修改工作节点的标签：

```shell
[root@master ~]# kubectl label node node1 node-role.kubernetes.io/worker=worker
node/node1 labeled
[root@master ~]# kubectl label node node2 node-role.kubernetes.io/worker=worker
node/node2 labeled
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   29m   v1.23.3
node1    NotReady   worker                 25m   v1.23.3
node2    NotReady   worker                 25m   v1.23.3
```

#### 六、安装kubernetes网络组件-Calico

上传calico.yaml到master上，使用yaml文件安装calico 网络插件

下载地址：https://docs.projectcalico.org/manifests/calico.yaml

```shell
[root@master ~]# kubectl apply -f  calico.yaml
[root@master ~]# kubectl get pod -n kube-system
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   38m   v1.23.3
node1    Ready    worker                 34m   v1.23.3
node2    Ready    worker                 34m   v1.23.3
```



