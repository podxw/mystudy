# Linux高级---k8s配置pv+pvc+nfs综合实验

[TOC]

## 一、搭建好nfs服务器

```shell
[root@nfs-server web]# yum install nfs-utils -y
[root@nfs-server web]# service nfs restart
```

## 二、设置共享目录

```shell
[root@nfs-server web]# vim /etc/exports
[root@nfs-server web]# cat /etc/exports
/sc/web   192.168.17.0/24(rw,sync)
[root@nfs-server web]# 
```

## 三、新建共享目录和index.html网页

```shell
[root@nfs-server web]# mkdir /sc/web/ -p
[root@nfs-server web]# cd /sc/web/
[root@nfs-server web]# echo "welcome to nongda" >index.html
[root@nfs-server web]# ls
index.html
[root@nfs-server web]# cat index.html 
welcome to nongda
```

## 四、刷新nfs或者重新输出共享目录

```shell
[root@nfs-server web]# exportfs -a   #输出所有共享目录
[root@nfs-server web]# exportfs -v   #显示输出的共享目录
/sc/web       	192.168.17.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[root@nfs-server web]# 
```

## 五、其他master和node节点

>每个node节点都需要安装nfs-utils工具，不然不能挂载nfs的共享目录

```shell
# 每个节点node包括master均需要进行下面的操作
# mater：
[root@master sanchuang]# mkdir /sanchuang/ -p
[root@master sanchuang]# mount 192.168.17.150:/sc/web /sanchuang
[root@master sanchuang]# df | grep web
192.168.17.150:/sc/web  17811456 3036416 14775040   18% /sanchuang
[root@master sanchuang]# 
[root@master sanchuang]# cat index.html 
welcome to nongda
[root@master sanchuang]# 

# node1:
[root@node1 ~]# mkdir /sanchuang
[root@node1 ~]# cd /sanchuang/
[root@node1 sanchuang]# ls
[root@node1 sanchuang]# mount 192.168.17.150:/sc/web /sanchuang
[root@node1 sanchuang]# df | grep web
192.168.17.150:/sc/web  17811456 3051008 14760448   18% /sanchuang

# node2 同样的操作
```

## 六、创建pv使用nfs服务器上的共享目录

```shell
[root@master pv-nfs]# cat pv-nfs.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sc-nginx-pv
  labels:
    type: sc-nginx-pv
spec:
  capacity:
    storage: 5Gi 
  accessModes:
    - ReadWriteMany
  storageClassName: nfs         #pv对应的名字
  nfs:
    path: "/sc/web"       #nfs共享的目录
    server: 192.168.17.150   #nfs服务器的ip地址
    readOnly: false   #访问模式
[root@master pv-nfs]# kubectl apply -f pv-nfs.yaml 
[root@master pv-nfs]# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                   STORAGECLASS   REASON   AGE
sc-nginx-pv      5Gi        RWX            Retain           Available                           nfs                     142m
```

## 七、创建pvc使用pv

```shell
[root@master pv-nfs]# cat pvc-nfs.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sc-nginx-pvc
spec:
  accessModes:
  - ReadWriteMany      
  resources:
     requests:
       storage: 1Gi
  storageClassName: nfs #使用nfs类型的pv
[root@master pv-nfs]# kubectl apply -f pvc-nfs.yaml 
persistentvolumeclaim/sc-nginx-pvc created
[root@master pv-nfs]# kubectl get pvc
NAME            STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
sc-nginx-pvc    Bound    sc-nginx-pv      5Gi        RWX            nfs            7s
```

## 八、创建pod使用pvc

```shell
[root@master pv-nfs]# cat pod-nfs.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: sc-pv-pod-nfs
spec:
  volumes:
    - name: sc-pv-storage-nfs
      persistentVolumeClaim:
        claimName: sc-nginx-pvc
  containers:
    - name: sc-pv-container-nfs
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: sc-pv-storage-nfs
[root@master pv-nfs]# kubectl apply -f pod-nfs.yaml 
pod/sc-pv-pod-nfs created
[root@master pv-nfs]# kubectl get pod
NAME                          READY   STATUS              RESTARTS   AGE
php-apache-7c97954b84-d9jng   1/1     Running             0          80m
sc-pv-pod-nfs                 0/1     ContainerCreating   0          4s
task-pv-pod                   1/1     Running             0          32m
[root@master pv-nfs]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
php-apache-7c97954b84-d9jng   1/1     Running   0          80m   10.244.2.202   node2   <none>           <none>
sc-pv-pod-nfs                 1/1     Running   0          10s   10.244.2.207   node2   <none>           <none>
task-pv-pod                   1/1     Running   0          32m   10.244.1.163   node1   <none>           <none>
[root@master pv-nfs]# 
```

## 九、测试

```shell
[root@master pv-nfs]# kubectl exec -it sc-pv-pod-nfs -- bash
root@sc-pv-pod-nfs:/# cd /usr/share/nginx/html/
root@sc-pv-pod-nfs:/usr/share/nginx/html# ls
index.html
root@sc-pv-pod-nfs:/usr/share/nginx/html# cat index.html 
welcome to nongda

[root@master pv-nfs]# curl 10.244.2.207
welcome to nongda
[root@master pv-nfs]# 
```

## 十、模拟网页更新

```shell
# 在nfs服务器上，修改index.html
[root@nfs-server web]# echo "123">> index.html 
[root@nfs-server web]# echo "123">> index.html 
[root@nfs-server web]# cat index.html 
welcome to nongda
123
123
[root@nfs-server web]# 

# 再次访问pod
[root@master pv-nfs]# curl 10.244.2.207
welcome to nongda
123
123
[root@master pv-nfs]# 
# nice 非常成功
```
