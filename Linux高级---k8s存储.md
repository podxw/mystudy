# Linux高级---k8s存储

[TOC]

## 一、数据卷的概述

>Kubernetes Volume（数据卷）主要解决了如下两方面问题：

- 数据持久性：通常情况下，容器运行起来之后，写入到其文件系统的文件暂时性的。当容器崩溃后，kubelet 将会重启该容器，此时原容器运行后写入的文件将丢失，因为容器将重新从镜像创建。
- 数据共享：同一个 Pod（容器组）中运行的容器之间，经常会存在共享文件/文件夹的需求

```markdown
    Docker 里同样也存在一个 volume（数据卷）的概念，但是 docker 对数据卷的管理相对 kubernetes 而言要更少一些。在 Docker 里，一个 Volume（数据卷）仅仅是宿主机（或另一个容器）文件系统上的一个文件夹。Docker 并不管理 Volume（数据卷）的生命周期。
```

```markdown
     在 Kubernetes 里，Volume（数据卷）存在明确的生命周期（与包含该数据卷的容器组相同）。因此，Volume（数据卷）的生命周期比同一容器组中任意容器的生命周期要更长，不管容器重启了多少次，数据都能被保留下来。当然，如果容器组退出了，数据卷也就自然退出了。此时，根据容器组所使用的 Volume（数据卷）类型不同，数据可能随数据卷的退出而删除，也可能被真正持久化，并在下次容器组重启时仍然可以使用。

     从根本上来说，一个 Volume（数据卷）仅仅是一个可被容器组中的容器访问的文件目录（也许其中包含一些数据文件）。这个目录是怎么来的，取决于该数据卷的类型（不同类型的数据卷使用不同的存储介质）。

     使用 Volume（数据卷）时，我们需要先在容器组中定义一个数据卷，并将其挂载到容器的挂载点上。容器中的一个进程所看到（可访问）的文件系统是由容器的 docker 镜像和容器所挂载的数据卷共同组成的。Docker 镜像将被首先加载到该容器的文件系统，任何数据卷都被在此之后挂载到指定的路径上。Volume（数据卷）不能被挂载到其他数据卷上，或者通过引用其他数据卷。同一个容器组中的不同容器各自独立地挂载数据卷，即同一个容器组中的两个容器可以将同一个数据卷挂载到各自不同的路径上。
```

## 二、关系图

>我们现在通过下图来理解 容器组、容器、挂载点、数据卷、存储介质（nfs、PVC、ConfigMap）等几个概念之间的关系

- 一个容器组可以包含多个数据卷、多个容器
- 一个容器通过挂载点决定某一个数据卷被挂载到容器中的什么路径
- 不同类型的数据卷对应不同的存储介质（图中列出了 nfs、PVC、ConfigMap 三种存储介质，接下来将介绍更多）

![image-20230530202523798](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530202523798.png)

> kubernetes的Volume支持多种类型，比较常见的有下面几个：

- 简单存储：EmptyDir、HostPath、NFS
- 高级存储：PV、PVC
- 配置存储：ConfigMap、Secret

## 三、数据卷的类型

### 1、emptydir

#### a、描述

```markdown
    emptyDir类型的数据卷在容器组被创建时分配给该容器组，并且直到容器组被移除，该数据卷才被释放。该数据卷初始分配时，始终是一个空目录。同一容器组中的不同容器都可以对该目录执行读写操作，并且共享其中的数据，（尽管不同的容器可能将该数据卷挂载到容器中的不同路径）。当容器组被移除时，emptyDir数据卷中的数据将被永久删除
```

>容器崩溃时，kubelet 并不会删除容器组，而仅仅是将容器重启，因此 emptyDir 中的数据在容器崩溃并重启后，仍然是存在的。

#### b、适用场景

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）

#### c、emptydir应用

> 接下来，通过一个容器之间文件共享的案例来使用一下EmptyDir。

```markdown
     在一个Pod中准备两个容器nginx和busybox，然后声明一个Volume分别挂在到两个容器的目录中，然后nginx容器负责向Volume中写日志，busybox中通过命令将日志内容读到控制台。
```

![image-20230530203215555](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530203215555.png)

>创建一个volume-emptydir.yaml
>
>![image-20230530204247159](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530204247159.png)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.14-alpine
    ports:
    - containerPort: 80
    volumeMounts:  # 将logs-volume挂在到nginx容器中，对应的目录为 /var/log/nginx
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件中内容
    volumeMounts:  # 将logs-volume 挂在到busybox容器中，对应的目录为 /logs
    - name: logs-volume
      mountPath: /logs
  volumes: # 声明volume， name为logs-volume，类型为emptyDir
  - name: logs-volume
    emptyDir: {}
```

```shell
# 创建pod
[root@master emptydir]# kubectl apply -f volume-emptydir.yaml 
pod/volume-emptydir created

# 查看pod
[root@master emptydir]# kubectl get pods -n dev -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
volume-emptydir   2/2     Running   0          53s   10.244.135.11   node3   <none>           <none>

# 通过podIp访问nginx
[root@master emptydir]# curl 10.244.135.11
......

# 通过kubectl logs命令查看指定容器的标准输出
[root@master emptydir]# kubectl logs -f volume-emptydir -c busybox -n dev
10.244.219.64 - - [30/May/2023:12:20:36 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
10.244.219.64 - - [30/May/2023:12:20:52 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

# 当我们删除这个pod时，创建的临时文件也会一并被删除
```

![image-20230530203623346](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530203623346.png)

![image-20230530203645698](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530203645698.png)

### 2、hostpath

#### a、描述

> hostPath 类型的数据卷将 Pod（容器组）所在节点的文件系统上某一个文件或文件夹挂载进容器组（容器）。
>
> **EmptyDir中数据不会被持久化，它会随着Pod的结束而销毁，如果想简单的将数据持久化到主机中，可以选择HostPath**。
>
> HostPath就是将Node主机中一个实际目录挂在到Pod中，以供容器使用，这样的设计就可以保证Pod销毁了，但是数据依据可以存在于Node主机上。

#### b、适用场景

绝大多数容器组并不需要使用 hostPath 数据卷，但是少数情况下，hostPath 数据卷非常有用：

- 某容器需要访问 Docker，可使用 hostPath 挂载宿主节点的 /var/lib/docker
- 在容器中运行 cAdvisor，使用 hostPath 挂载宿主节点的 /sys

#### c、hostpath应用

![image-20230530204146734](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530204146734.png)

>创建一个volume-hostpath.yaml：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    hostPath: 
      path: /root/logs
      type: DirectoryOrCreate  # 目录存在就使用，不存在就先创建后使用
```

```markdown
关于type的值的一点说明：
	DirectoryOrCreate 目录存在就使用，不存在就先创建后使用
	Directory	目录必须存在
	FileOrCreate  文件存在就使用，不存在就先创建后使用
	File 文件必须存在	
    Socket	unix套接字必须存在
	CharDevice	字符设备必须存在
	BlockDevice 块设备必须存在
```

```shell
# 创建Pod
[root@master volume]# kubectl create -f volume-hostpath.yaml
pod/volume-hostpath created

# 查看Pod
[root@master volume]# kubectl get pod -n dev -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
volume-hostpath   2/2     Running   0          91s   10.244.135.12   node3   <none>           <none>

#访问nginx
[root@master volume]# curl 10.244.135.12

# 接下来就可以去host的/root/logs目录下查看存储的文件了
###  注意: 下面的操作需要到Pod所在的节点运行（案例中是node3）
[root@node3 ~]# ls /root/logs/
access.log  error.log

# 同样的道理，如果在此目录下创建一个文件，到容器中也是可以看到的

# 当我们删除这个pod时，创建的logs文件不会删除
[root@master volume]# kubectl delete -f volume-hostpath.yaml 
pod "volume-hostpath" deleted

# 进入node3查看，发现文件还是存在
[root@node3 logs]# ls
access.log  error.log
[root@node3 logs]# 
```

### 3、nfs

#### a、描述

>HostPath可以解决数据持久化的问题，但是一旦Node节点故障了，Pod如果转移到了别的节点，又会出现问题了，此时需要准备单独的网络存储系统，比较常用的用NFS、CIFS。
>
>NFS是一个网络文件存储系统，可以搭建一台NFS服务器，然后将Pod中的存储直接连接到NFS系统上，这样的话，无论Pod在节点上怎么转移，只要Node跟NFS的对接没问题，数据就可以成功访问。

![image-20230530210129234](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230530210129234.png)

#### b、适用场景

- 存储日志文件
- MySQL的data目录（建议只在测试环境中）
- 用户上传的临时文件

#### c、nfs应用

1）首先要准备nfs的服务器

```shell
# 在master上安装nfs服务
[root@nfs-server ~]# yum install nfs-utils -y

# 准备一个共享目录
[root@nfs-server ~]# mkdir /root/data/nfs/ -pv

# 将共享目录以读写权限暴露给192.168.17.0/24网段中的所有主机
[root@nfs-server ~]# vim /etc/exports
[root@nfs-server ~]# cat /etc/exports
/root/data/nfs 192.168.17.0/24(rw,no_root_squash,no_all_squash,sync)
 
# 启动nfs服务
[root@nfs-server ~]# systemctl start nfs
```

2）接下来，要在的每个node节点上都安装下nfs，这样的目的是为了node节点可以驱动nfs设备

```powershell
# 在node上安装nfs服务，注意不需要启动
[root@master ~]# yum install nfs-utils -y
```

3）接下来，就可以编写pod的配置文件了，创建volume-nfs.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-nfs
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] 
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    nfs:
      server: 172.16.17.146  #nfs服务器地址
      path: /root/data/nfs #共享文件路径
```

4）最后，运行下pod，观察结果

```shell
# 创建pod
[root@master volume]# kubectl create -f volume-nfs.yaml
pod/volume-nfs created

# 查看pod
[root@master volume]# kubectl get pod -n dev -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
volume-nfs   2/2     Running   0          4s    10.244.135.13   node3   <none>           <none>

# 查看nfs服务器上的共享目录，发现已经有文件了，说明成功挂载
[root@nfs-server ~]# ll /root/data/nfs/
总用量 0
-rw-r--r--. 1 root root 0 5月  30 21:23 access.log
-rw-r--r--. 1 root root 0 5月  30 21:23 error.log
[root@nfs-server ~]# 

# 测试访问，即可在nfs服务器上看到结果
[root@master volume]# curl 10.244.135.13
...

# 在nfs服务器上查看
[root@nfs-server nfs]# tail -f access.log 
10.244.219.64 - - [30/May/2023:13:25:49 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
```

### 4、PV和PVC

#### a、描述

```markdown
    与管理计算资源相比，管理存储资源是一个完全不同的问题。为了更好的管理存储，Kubernetes 引入了 PersistentVolume 和 PersistentVolumeClaim 两个概念，将存储管理抽象成如何提供存储以及如何使用存储两个关注点。
```

>PersistentVolume（PV 存储卷）是集群中的一块存储空间，由集群管理员管理、或者由 Storage Class（存储类）自动管理。PV（存储卷）和 node（节点）一样，是集群中的资源（kubernetes 集群由存储资源和计算资源组成）。PersistentVolumeClaim（存储卷声明）是一种类型的 Volume（数据卷），PersistentVolumeClaim（存储卷声明）引用的 PersistentVolume（存储卷）有自己的生命周期，该生命周期独立于任何使用它的容器组。PersistentVolume（存储卷）描述了如何提供存储的细节信息（NFS、cephfs等存储的具体参数）。
>
>PersistentVolumeClaim（PVC 存储卷声明）代表用户使用存储的请求。Pod 容器组消耗 node 计算资源，PVC 存储卷声明消耗 PersistentVolume 存储资源。Pod 容器组可以请求特定数量的计算资源（CPU / 内存）；PersistentVolumeClaim 可以请求特定大小/特定访问模式（只能被单节点读写/可被多节点只读/可被多节点读写）的存储资源。
>
>根据应用程序的特点不同，其所需要的存储资源也存在不同的要求，例如读写性能等。集群管理员必须能够提供关于 PersistentVolume（存储卷）的更多选择，无需用户关心存储卷背后的实现细节。为了解决这个问题，Kubernetes 引入了 StorageClass（存储类）的概念

#### b、存储卷和存储卷声明的关系

> 存储卷和存储卷声明的关系如下图所示：

- PersistentVolume 是集群中的存储资源，通常由集群管理员创建和管理
- StorageClass 用于对 PersistentVolume 进行分类，如果正确配置，StorageClass 也可以根据 PersistentVolumeClaim 的请求动态创建 Persistent Volume
- PersistentVolumeClaim 是使用该资源的请求，通常由应用程序提出请求，并指定对应的 StorageClass 和需求的空间大小
- PersistentVolumeClaim 可以做为数据卷的一种，被挂载到容器组/容器中使用

![image-20230531192930728](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230531192930728.png)

#### c、存储卷声明的管理过程

PersistantVolume 和 PersistantVolumeClaim 的管理过程描述如下：

> 下图主要描述的是 PV 和 PVC 的管理过程，因为绘制空间的问题，将挂载点与Pod关联了，实际结构应该如上图所示：
>
> - Pod 中添加数据卷，数据卷关联PVC
> - Pod 中包含容器，容器挂载数据卷

![image-20230531193010336](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230531193010336.png)

### 5、PV

#### a、资源清单

>PV是存储资源的抽象，下面是资源清单文件:

```yaml
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，与底层真正存储对应
  capacity:  # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes:  # 访问模式
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
```

PV 的关键配置参数说明：

- **存储类型**

  底层实际存储的类型，kubernetes支持多种存储类型，每种存储类型的配置都有所差异

- **存储能力（capacity）**

​      目前只支持存储空间的设置( storage=1Gi )，不过未来可能会加入IOPS、吞吐量等指标的配置

- **访问模式（accessModes）**

  用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：

  - ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
  - ReadOnlyMany（ROX）：  只读权限，可以被多个节点挂载
  - ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

  `需要注意的是，底层不同的存储类型可能支持的访问模式不同`

- **回收策略（persistentVolumeReclaimPolicy）**

  当PV不再被使用了之后，对其的处理方式。目前支持三种策略：

  - Retain  （保留）  保留数据，需要管理员手工清理数据
  - Recycle（回收）  清除 PV 中的数据，效果相当于执行 rm -rf /thevolume/*
  - Delete  （删除） 与 PV 相连的后端存储完成 volume 的删除操作，当然这常见于云服务商的存储服务

  `需要注意的是，底层不同的存储类型可能支持的回收策略不同`

- **存储类别**

  PV可以通过storageClassName参数指定一个存储类别

  - 具有特定类别的PV只能与请求了该类别的PVC进行绑定
  - 未设定类别的PV则只能与不请求任何类别的PVC进行绑定

- **状态（status）**

  一个 PV 的生命周期中，可能会处于4中不同的阶段：

  - Available（可用）：     表示可用状态，还未被任何 PVC 绑定
  - Bound（已绑定）：     表示 PV 已经被 PVC 绑定
  - Released（已释放）： 表示 PVC 被删除，但是资源还未被集群重新声明
  - Failed（失败）：         表示该 PV 的自动回收失败

#### b、pv应用

使用NFS作为存储，来演示PV的使用，创建3个PV，对应NFS中的3个暴露的路径。

1) 准备NFS环境

```shell
# 创建目录
[root@nfs-server ~]# mkdir /root/data/{pv1,pv2,pv3} -pv

# 暴露服务
[root@nfs-server ~]# more /etc/exports
/root/data/pv1     192.168.17.0/24(rw,no_root_squash,no_all_squash,sync)
/root/data/pv2     192.168.17.0/24(rw,no_root_squash,no_all_squash,sync)
/root/data/pv3     192.168.17.0/24(rw,no_root_squash,no_all_squash,sync)

# 重启服务
[root@nfs-server ~]# systemctl restart nfs
```

2) 创建pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv1
spec:
  capacity: 
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv1
    server: 192.168.17.146

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv2
spec:
  capacity: 
    storage: 2Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv2
    server: 192.168.17.146
    
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv3
spec:
  capacity: 
    storage: 3Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv3
    server: 192.168.17.146
```

```powershell
# 创建 pv
[root@master pv-pvc]# kubectl create -f pv.yaml 
persistentvolume/pv1 created
persistentvolume/pv2 created
persistentvolume/pv3 created

# 查看 pv
[root@master pv-pvc]# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv1    1Gi        RWX            Retain           Available                                   6s
pv2    2Gi        RWX            Retain           Available                                   6s
pv3    3Gi        RWX            Retain           Available                                   6s
[root@master pv-pvc]# 
```

### 6、PVC

#### a、资源清单

> PVC是资源的申请，用来声明对存储空间、访问模式、存储类别需求信息。下面是资源清单文件:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  namespace: dev
spec:
  accessModes: # 访问模式
  selector: # 采用标签对PV选择
  storageClassName: # 存储类别
  resources: # 请求空间
    requests:
      storage: 5Gi
```

PVC 的关键配置参数说明：

- **访问模式（accessModes）**

​       用于描述用户应用对存储资源的访问权限

- **选择条件（selector）**

  通过Label Selector的设置，可使PVC对于系统中己存在的PV进行筛选

- **存储类别（storageClassName）**

  PVC在定义时可以设定需要的后端存储的类别，只有设置了该class的pv才能被系统选出

- **资源请求（Resources ）**

  描述对存储资源的请求

#### b、pvc应用

1)  创建pvc.yaml，申请pv

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
      
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
     
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc3
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```

```powershell
# 创建pvc
[root@master pv-pvc]# kubectl create -f pvc.yaml 
persistentvolumeclaim/pvc1 created
persistentvolumeclaim/pvc2 created
persistentvolumeclaim/pvc3 created

# 查看pvc
[root@master pv-pvc]# kubectl get pvc -n dev -o wide
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE   VOLUMEMODE
pvc1   Bound    pv1      1Gi        RWX                           20s   Filesystem
pvc2   Bound    pv2      2Gi        RWX                           20s   Filesystem
pvc3   Bound    pv3      3Gi        RWX                           20s   Filesystem

# 查看pv
[root@master pv-pvc]# kubectl get pv -o wide
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM      STORAGECLASS   REASON   AGE     VOLUMEMODE
pv1    1Gi        RWX            Retain           Bound    dev/pvc1                           5m11s   Filesystem
pv2    2Gi        RWX            Retain           Bound    dev/pvc2                           5m11s   Filesystem
pv3    3Gi        RWX            Retain           Bound    dev/pvc3                           5m11s   Filesystem
```

2)  创建pods.yaml, 使用pv

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc1
        readOnly: false
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod2 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc2
        readOnly: false        
---
apiVersion: v1
kind: Pod
metadata:
  name: pod3
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod3 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc3
        readOnly: false   
```

```powershell
# 创建pod
[root@master pv-pvc]# kubectl create -f pod.yaml 
pod/pod1 created
pod/pod2 created
pod/pod3 created

# 查看pod
[root@master pv-pvc]# kubectl get pod -n dev -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
pod1              1/1     Running   0          51s     10.244.104.29    node2   <none>           <none>
pod2              1/1     Running   0          51s     10.244.166.141   node1   <none>           <none>
pod3              1/1     Running   0          51s     10.244.104.28    node2   <none>           <none>
[root@master pv-pvc]# 

# 查看pvc
[root@master pv-pvc]# kubectl get pvc -n dev -o wide
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE     VOLUMEMODE
pvc1   Bound    pv1      1Gi        RWX                           6m25s   Filesystem
pvc2   Bound    pv2      2Gi        RWX                           6m25s   Filesystem
pvc3   Bound    pv3      3Gi        RWX                           6m25s   Filesystem

# 查看pv
[root@master pv-pvc]# kubectl get pv -o wide
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM      STORAGECLASS   REASON   AGE   VOLUMEMODE
pv1    1Gi        RWX            Retain           Bound    dev/pvc1                           11m   Filesystem
pv2    2Gi        RWX            Retain           Bound    dev/pvc2                           11m   Filesystem
pv3    3Gi        RWX            Retain           Bound    dev/pvc3                           11m   Filesystem

# 查看nfs中的文件存储
[root@nfs-server ~]# cd /root/data/
[root@nfs-server data]# ls
nfs  pv1  pv2  pv3
[root@nfs-server pv1]# cat out.txt 
pod1
pod1
....

[root@nfs-server pv2]# cat out.txt 
pod2
pod2
....

[root@nfs-server pv3]# cat out.txt 
pod3
pod3
....
```

## 三、生命周期

### 1、生命周期的概念

PVC和PV是一一对应的，PV和PVC之间的相互作用遵循以下生命周期：

- **资源供应**：管理员手动创建底层存储和PV

- **资源绑定**：用户创建PVC，kubernetes负责根据PVC的声明去寻找PV，并绑定

  在用户定义好PVC之后，系统将根据PVC对存储资源的请求在已存在的PV中选择一个满足条件的

  - 一旦找到，就将该PV与用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了
  - 如果找不到，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV

  PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了

- **资源使用**：用户可在pod中像volume一样使用pvc

  Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。

- **资源释放**：用户删除pvc来释放pv

  当存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用。

- **资源回收**：kubernetes根据pv设置的回收策略进行资源的回收

  对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用

![image-20230531210717962](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230531210717962.png)

```powershell
# 当我们释放pod和pvc时，此时的pv进入了Released状态，这个时候是不能够被使用的
# 因为我们设置的回收策略为Retain  （保留）  保留数据，需要管理员手工清理数据
[root@master pv-pvc]# kubectl delete -f pod.yaml 
pod "pod1" deleted
pod "pod2" deleted
pod "pod3" deleted
[root@master pv-pvc]# kubectl delete -f pvc.yaml 
persistentvolumeclaim "pvc1" deleted
persistentvolumeclaim "pvc2" deleted
persistentvolumeclaim "pvc3" deleted
[root@master pv-pvc]# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM      STORAGECLASS   REASON   AGE
pv1    1Gi        RWX            Retain           Released   dev/pvc1                           5m49s
pv2    2Gi        RWX            Retain           Released   dev/pvc2                           5m49s
pv3    3Gi        RWX            Retain           Released   dev/pvc3                           5m49s
```

### 2、恢复重新绑定

![image-20230531211147043](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---k8s%E5%AD%98%E5%82%A8.assets%5Cimage-20230531211147043.png)

```powershell
[root@master pv-pvc]# kubectl edit pv pv1
persistentvolume/pv1 edited
[root@master pv-pvc]# kubectl edit pv pv2
persistentvolume/pv2 edited
[root@master pv-pvc]# kubectl edit pv pv3
persistentvolume/pv3 edited
[root@master pv-pvc]# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv1    1Gi        RWX            Retain           Available                                   7m24s
pv2    2Gi        RWX            Retain           Available                                   7m24s
pv3    3Gi        RWX            Retain           Available                                   7m24s
[root@master pv-pvc]# 
```

## 四、配置存储

### 1、configmap

ConfigMap 提供了一种向容器组注入配置信息的途径。ConfigMap 中的数据可以被 Pod（容器组）中的容器作为一个数据卷挂载。

在数据卷中引用 ConfigMap 时：

- 您可以直接引用整个 ConfigMap 到数据卷，此时 ConfigMap 中的每一个 key 对应一个文件名，value 对应该文件的内容
- 您也可以只引用 ConfigMap 中的某一个名值对，此时可以将 key 映射成一个新的文件名

#### a、创建configmap

1）创建configmap.yaml，内容如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
  namespace: dev
data:
  info: |
    username:admin
    password:123456
```

2）接下来，使用此配置文件创建configmap

```powershell
# 创建comfigmap
[root@master configmap]# kubectl create -f configmap.yaml 
configmap/configmap created

# 查看configmap的情况
[root@master configmap]# kubectl get cm -n dev
NAME               DATA   AGE
configmap          1      23s
kube-root-ca.crt   1      25h
[root@master configmap]# kubectl describe cm configmap -n dev
Name:         configmap
Namespace:    dev
Labels:       <none>
Annotations:  <none>

Data
====
info:
----
username:admin
password:123456


BinaryData
====

Events:  <none>
[root@master configmap]# 
```

3）接下来创建一个pod-configmap.yaml，将上面创建的configmap挂载进去

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将configmap挂载到目录
    - name: config
      mountPath: /configmap/config
  volumes: # 引用configmap
  - name: config
    configMap:
      name: configmap
```

```powershell
# 创建pod
[root@master configmap]# kubectl apply -f pod-comfigmap.yaml 
pod/pod-configmap created

# 查看pod
[root@master configmap]# kubectl get pod -n dev -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
pod-configmap     1/1     Running   0          53s     10.244.104.32    node2   <none>           <none>

# 进入容器
[root@master configmap]# kubectl exec -it pod-configmap -n dev -- bash
root@pod-configmap:/# cd /configmap/config/
root@pod-configmap:/configmap/config# ls
info
root@pod-configmap:/configmap/config# cat info 
username:admin
password:123456
root@pod-configmap:/configmap/config# 

# 可以看到映射已经成功，每个configmap都映射成了一个目录
# key--->文件     value---->文件中的内容
# 此时如果更新configmap的内容, 容器中的值也会动态更新
```

#### b、另外一种创建方式

```powershell
# 从文件创建configmap
[root@master configmap]# echo 1234 > index.html
[root@master configmap]# kubectl create configmap web-config  --from-file=index.html 
configmap/web-config created

# 查看configmap
[root@master configmap]# kubectl get cm
NAME               DATA   AGE
web-config         1      7s
[root@master configmap]# kubectl describe cm web-config
Name:         web-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
index.html:
----
1234


BinaryData
====

Events:  <none>

# 创建pod使用该configmap
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将configmap挂载到目录
    - name: config
      mountPath: /usr/share/nginx/html
  volumes: # 引用configmap
  - name: config
    configMap:
      name: web-config
```

```powershell
# 创建pod
[root@master configmap]# kubectl create -f pod2-configmap.yaml 
pod/pod-configmap created

# 查看pod
[root@master configmap]# kubectl get  pod -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
pod-configmap   1/1     Running   0          75s   10.244.104.33   node2   <none>           <none>

# 测试访问
[root@master configmap]# curl 10.244.104.33
1234
[root@master configmap]# 
```

### 2、secret

​    在kubernetes中，还存在一种和ConfigMap非常类似的对象，称为Secret对象。它主要用于存储敏感信息，例如密码、秘钥、证书等等。

1)  首先使用base64对数据进行编码

```powershell
[root@master secret]# echo 'admin' | base64
YWRtaW4K
[root@master secret]# echo -n '123456' | base64
MTIzNDU2
```

2)  接下来编写secret.yaml，并创建Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
data:
  username: YWRtaW4K
  password: MTIzNDU2
```

```powershell
# 创建secret
[root@master secret]# kubectl create -f secret.yaml
secret/secret created

# 查看secret详情
[root@master secret]# kubectl describe secret secret -n dev
Name:         secret
Namespace:    dev
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
username:  6 bytes
[root@master secret]# kubectl get secret -n dev
NAME                  TYPE                                  DATA   AGE
secret                Opaque                                2      47s
[root@master secret]# 
```

3) 创建pod-secret.yaml，将上面创建的secret挂载进去：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将secret挂载到目录
    - name: config
      mountPath: /secret/config
  volumes:
  - name: config
    secret:
      secretName: secret
```

```powershell
# 创建pod
[root@master secret]# kubectl create -f pod-secret.yaml 
pod/pod-secret created

# 查看pod
[root@master secret]# kubectl get pod -n dev
NAME              READY   STATUS    RESTARTS   AGE
pod-secret        1/1     Running   0          8s

# 进入容器，查看secret信息，发现已经自动解码了
[root@master secret]# kubectl exec -it  pod-secret -n dev -- bash
root@pod-secret:/# cd /secret/config/
root@pod-secret:/secret/config# ls
password  username
root@pod-secret:/secret/config# cat password 
123456
root@pod-secret:/secret/config# cat username 
admin
```

至此，已经实现了利用secret实现了信息的编码。