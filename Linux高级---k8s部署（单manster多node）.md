# Linux高级---k8s部署（单manster多node）

[TOC]

## 一、实验环境

```
一个master服务器，三台node服务器
软件： centos7.9  docker
硬件： 2G/2C
我们采用kubeadm方式来安装
```

## 二、实验步骤

### 1、机器的基本准备

>修改主机的hostname，设置静态IP
>
>192.168.17.144 scmaster
>192.168.17.148 scnode1
>192.168.17.149 scnode2
>192.168.17.150 scnode3

### 2、关闭selinux和firewalld

**每一台服务器都需要完成**

```shell
[root@scmaster ~]# service firewalld stop
Redirecting to /bin/systemctl stop firewalld.service
[root@scmaster ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@scmaster ~]#
[root@scmaster ~]# vim /etc/selinux/config 
SELINUX=disabled
[root@scmaster ~]# getenforce 
Enforcing
[root@scmaster ~]# setenforce 0
[root@scmaster ~]# 
```

### 3、升级所有的软件

**每一台服务器都需要完成**

```shell
yum update -y
```

### 4、安装docker

**每一台服务器都需要完成**

```shell
1.卸载原来安装过的docker，如果没有安装可以不需要卸载
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
2.安装yum相关的工具，下载docker-ce.repo文件
[root@cali ~]# 
[root@cali ~]#  yum install -y yum-utils -y
 [root@cali ~]#yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
下载docker-ce.repo文件存放在/etc/yum.repos.d
[root@cali yum.repos.d]# pwd
/etc/yum.repos.d
[root@cali yum.repos.d]# ls
CentOS-Base.repo  CentOS-Debuginfo.repo  CentOS-Media.repo    CentOS-Vault.repo          docker-ce.repo
CentOS-CR.repo    CentOS-fasttrack.repo  CentOS-Sources.repo  CentOS-x86_64-kernel.repo  nginx.repo
[root@cali yum.repos.d]# 
3.安装docker-ce软件
container engine 容器引擎
docker是一个容器管理的软件
docker-ce 是服务器端软件 server
docker-ce-cli 是客户端软件 client
docker-compose-plugin 是compose插件，用来批量启动很多容器，在单台机器上
containerd.io  底层用来启动容器的
[root@cali yum.repos.d]# yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

[root@scmaster ~]# docker --version
Docker version 20.10.18, build b40c2f6
[root@scmaster ~]# 
4.启动docker服务

[root@scmaster ~]# systemctl start docker
[root@scmaster ~]# 
[root@scmaster ~]# ps aux|grep docker
root      53288  1.5  2.3 1149960 43264 ?       Ssl  15:11   0:00 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root      53410  0.0  0.0 112824   984 pts/0    S+   15:11   0:00 grep --color=auto docker
[root@scmaster ~]# 
5.设置docker服务开机启动
[root@scmaster ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@scmaster ~]# 
```

### 5、配置 Docker使用systemd作为默认Cgroup驱动

每台服务器上都要操作，master和node上都要操作执行下面的脚本，会产生 /etc/docker/daemon.json文件

```shell

	cat <<EOF > /etc/docker/daemon.json
	{
	   "exec-opts": ["native.cgroupdriver=systemd"]
	}
	EOF
	
#重启docker
[root@scmaster docker]# systemctl restart docker
[root@scmaster docker]#
```

### 6、关闭swap分区

**每一台服务器都需要操作**

```shell
swapoff -a # 临时关闭
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab #永久关闭
```

### 7、修改hosts文件

**每一台服务器都需要操作**

```shell
[root@scmaster docker]# cat >> /etc/hosts << EOF 
> 192.168.227.130 scmaster
> 192.168.227.132 scnode1
> 192.168.227.133 scnode2
> 192.168.227.134 scnode3
> EOF
[root@scmaster docker]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.227.130 scmaster
192.168.227.132 scnode1
192.168.227.133 scnode2
192.168.227.134 scnode3
[root@scmaster docker]# 
```

### 8、修改内核参数

**每台机器上（master和node），永久修改  追加到内核会读取的参数文件里**

```shell
[root@scmaster docker]# cat <<EOF >>/etc/sysctl.conf 
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> net.ipv4.ip_nonlocal_bind = 1
> net.ipv4.ip_forward = 1
> vm.swappiness=0
> EOF
[root@scmaster docker]# cat /etc/sysctl.conf 
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
[root@scmaster docker]# 
 让内核重新读取数据，加载生效
[root@scmaster docker]# sysctl -p
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
[root@scmaster docker]#
```

### 9、安装kubeadm,kubelet和kubectl

**集群里的每台服务器都需要安装**

```shell
# 添加kubernetes YUM软件源
[root@scmaster docker]#	cat > /etc/yum.repos.d/kubernetes.repo << EOF
	[kubernetes]
	name=Kubernetes
	baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
	enabled=1
	gpgcheck=0
	repo_gpgcheck=0
	gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
	EOF
[root@scmaster docker]#yum install -y kubelet-1.20.6 kubeadm-1.20.6 kubectl-1.20.6  --》最好指定版本，因为1.24的版本默认的容器运行时环境不是docker了
[root@scmaster docker]#systemctl enable  kubelet
```

### 10、部署Kubernetes Master

**只在master节点上执行**

```shell
[root@master ~]#  docker pull  coredns/coredns:1.8.4

[root@master ~]# docker tag coredns/coredns:1.8.4 registry.aliyuncs.com/google_containers/coredns:v1.8.4

#初始化操作在master服务器上执行
[root@master ~]#kubeadm init \
	--apiserver-advertise-address=192.168.17.200 \
	--image-repository registry.aliyuncs.com/google_containers \
	--service-cidr=10.1.0.0/16 \
	--pod-network-cidr=10.244.0.0/16
	#192.168.227.130 是master的ip
	#      --service-cidr string                  Use alternative range of IP address for service VIPs. (default "10.96.0.0/12")  服务发布暴露--》dnat
	#      --pod-network-cidr string              Specify range of IP addresses for the pod network. If set, the control plane will automatically allocate CIDRs for every node.
[root@scmaster docker]# kubeadm init \
> --apiserver-advertise-address=192.168.17.144 \
> --image-repository registry.aliyuncs.com/google_containers \
> --service-cidr=10.1.0.0/16 \
> --pod-network-cidr=10.244.0.0/16
I0922 16:43:44.828548   53894 version.go:255] remote version is much newer: v1.25.2; falling back to: stable-1.23
[init] Using Kubernetes version: v1.23.12
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local scmaster] and IPs [10.1.0.1 192.168.227.130]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost scmaster] and IPs [192.168.227.130 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost scmaster] and IPs [192.168.227.130 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 12.008541 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node scmaster as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node scmaster as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: ykssm9.g8yfv9rd6avseqnu
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

kubeadm join 192.168.17.144:6443 --token ykssm9.g8yfv9rd6avseqnu \
	--discovery-token-ca-cert-hash sha256:e5a34c30d082042dfa876249123502dcddfde7e0695934d70fc41b37889fa0e2 
[root@scmaster docker]# 

完成初始化的新建文件和目录的操作，在master上完成
[root@scmaster docker]#   mkdir -p $HOME/.kube
[root@scmaster docker]#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@scmaster docker]#   sudo chown $(id -u):$(id -g) $HOME/.kube/config
[root@scmaster docker]# 
```

### 11、node节点服务器加入k8s集群

```shell
[root@scnode1 docker]# kubeadm join 192.168.227.130:6443 --token ykssm9.g8yfv9rd6avseqnu \
> --discovery-token-ca-cert-hash sha256:e5a34c30d082042dfa876249123502dcddfde7e0695934d70fc41b37889fa0e2 
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

[root@scnode1 docker]# 

所有的node节点都需要加入到k8s集群里，查看集群里的机器
[root@scmaster docker]# kubectl get node
NAME       STATUS     ROLES                  AGE     VERSION
scmaster   NotReady   control-plane,master   6m38s   v1.23.6
scnode1    NotReady   <none>                 2m22s   v1.23.6
scnode2    NotReady   <none>                 14s     v1.23.6
scnode3    NotReady   <none>                 7s      v1.23.6
[root@scmaster docker]# 

NotReady 说明master和node节点之间的通信还是有问题的，容器之间通信还没有准备好
```

### 12、安装网络插件flannel(在master节点执行)

**实现master上的pod和node节点上的pod之间通信**

```shell
[root@master ~]# cat kube-flannel.yml 
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-flannel
---    
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.19.2 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.2
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.19.2 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.2
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
[root@master ~]# 
[root@scmaster flannel]# kubectl apply -f kube-flannel.yml 
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
[root@scmaster flannel]# 

[root@scmaster flannel]# kubectl get node
NAME       STATUS   ROLES                  AGE   VERSION
scmaster   Ready    control-plane,master   12h   v1.23.6
scnode1    Ready    <none>                 12h   v1.23.6
scnode2    Ready    <none>                 12h   v1.23.6
scnode3    Ready    <none>                 12h   v1.23.6
```
