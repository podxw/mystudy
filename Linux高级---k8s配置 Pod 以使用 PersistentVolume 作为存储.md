# Linux高级---k8s配置 Pod 以使用 PersistentVolume 作为存储

[TOC]

>官方文档:https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

## 一、新建文件

>在master和所有的node节点上新建文件夹/mnt/data，同时在里面新建首页文件index.html

```shell
[root@master ~]# mkdir /mnt/data    ---》master上可以不新建，因为不会启动业务pod
[root@node3 ~]# mkdir  /mnt/data
[root@node2 ~]# mkdir  /mnt/data
[root@node1 ~]# mkdir  /mnt/data

[root@master ~]#  sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
[root@node2 ~]#  sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
[root@node3 ~]#  sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
[root@node1 ~]#  sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
```

```shell
# 在一台机器上查看即可
[root@node2 data]# cat index.html 
Hello from Kubernetes storage
[root@node2 data]# ls
index.html
[root@node2 data]# pwd
/mnt/data
[root@node2 data]# 
```

## 二、创建PersistentVolume

>创建 PersistentVolume，创建pv-volume.yaml文件，内容如下：  下面是官方的样例

```shell
[root@master pv]#  wget https://k8s.io/examples/pods/storage/pv-volume.yaml --no-check-certificate
--2023-03-12 16:43:01--  https://k8s.io/examples/pods/storage/pv-volume.yaml
正在解析主机 k8s.io (k8s.io)... 34.107.204.206, 2600:1901:0:26f3::
正在连接 k8s.io (k8s.io)|34.107.204.206|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 301 Moved Permanently
位置：https://kubernetes.io/examples/pods/storage/pv-volume.yaml [跟随至新的 URL]
--2023-03-12 16:43:02--  https://kubernetes.io/examples/pods/storage/pv-volume.yaml
正在解析主机 kubernetes.io (kubernetes.io)... 147.75.40.148
正在连接 kubernetes.io (kubernetes.io)|147.75.40.148|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：229 [application/x-yaml]
正在保存至: “pv-volume.yaml”

100%[=============================================================================================>] 229         --.-K/s 用时 0s      

2023-03-12 16:43:04 (22.8 MB/s) - 已保存 “pv-volume.yaml” [229/229])

[root@master pv]# ls
pv-volume.yaml
[root@master pv]# cat pv-volume.yaml  # 修改访问策略为ReadWriteMany
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual  #  ---》pvc通过这个存储类型的名字和pv绑定
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/data"
[root@master pv]# 
```

```shell
[root@master pv]# kubectl apply -f pv-volume.yaml 
persistentvolume/task-pv-volume created
[root@master pv]# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
task-pv-volume   10Gi       RWX            Retain           Available           manual                  11s
[root@master pv]# 
```

##  三、创建 PersistentVolumeClaim

```shell
[root@master pv]# wget https://k8s.io/examples/pods/storage/pv-claim.yaml --no-check-certificate
--2023-03-12 16:47:09--  https://k8s.io/examples/pods/storage/pv-claim.yaml
正在解析主机 k8s.io (k8s.io)... 34.107.204.206, 2600:1901:0:26f3::
正在连接 k8s.io (k8s.io)|34.107.204.206|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 301 Moved Permanently
位置：https://kubernetes.io/examples/pods/storage/pv-claim.yaml [跟随至新的 URL]
--2023-03-12 16:47:09--  https://kubernetes.io/examples/pods/storage/pv-claim.yaml
正在解析主机 kubernetes.io (kubernetes.io)... 147.75.40.148
正在连接 kubernetes.io (kubernetes.io)|147.75.40.148|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：189 [application/x-yaml]
正在保存至: “pv-claim.yaml”

100%[=============================================================================================>] 189         --.-K/s 用时 0s      

2023-03-12 16:47:10 (22.3 MB/s) - 已保存 “pv-claim.yaml” [189/189])

[root@master pv]# ls
pv-claim.yaml  pv-volume.yaml
[root@master pv]# vim pv-claim.yaml  # 修改pv访问策略为ReadWriteMany
[root@master pv]# cat pv-claim.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
[root@master pv]# 
```

```shell
[root@master pv]# kubectl apply -f pv-claim.yaml 
persistentvolumeclaim/task-pv-claim created
[root@master pv]# kubectl get pvc
NAME            STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pv-claim   Bound    task-pv-volume   10Gi       RWX            manual         5s
[root@master pv]# 
```

## 四、创建 Pod使用pvc

>存储使用流程:pod-->volume-->pvc-->pv

```shell
[root@master pv]# wget https://k8s.io/examples/pods/storage/pv-pod.yaml --no-check-certificate
--2023-03-12 16:49:42--  https://k8s.io/examples/pods/storage/pv-pod.yaml
正在解析主机 k8s.io (k8s.io)... 34.107.204.206, 2600:1901:0:26f3::
正在连接 k8s.io (k8s.io)|34.107.204.206|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 301 Moved Permanently
位置：https://kubernetes.io/examples/pods/storage/pv-pod.yaml [跟随至新的 URL]
--2023-03-12 16:49:43--  https://kubernetes.io/examples/pods/storage/pv-pod.yaml
正在解析主机 kubernetes.io (kubernetes.io)... 147.75.40.148
正在连接 kubernetes.io (kubernetes.io)|147.75.40.148|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：395 [application/x-yaml]
正在保存至: “pv-pod.yaml”

100%[=============================================================================================>] 395         --.-K/s 用时 0s      

2023-03-12 16:49:44 (33.9 MB/s) - 已保存 “pv-pod.yaml” [395/395])

[root@master pv]# cat pv-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage


[root@master pv]# 
```

```shell
[root@master pv]# kubectl apply -f pv-pod.yaml 
pod/task-pv-pod created
[root@master pv]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
php-apache-7c97954b84-d9jng   1/1     Running   0          48m   10.244.2.202   node2   <none>           <none>
task-pv-pod                   1/1     Running   0          15s   10.244.1.163   node1   <none>           <none>
[root@master pv]# 
```

## 五、验证pod是否使用pv里的数据

```shell
[root@master pv]# kubectl exec -it task-pv-pod -- bash
root@task-pv-pod:/# cd /usr/share/nginx/html/
root@task-pv-pod:/usr/share/nginx/html# ls
index.html
root@task-pv-pod:/usr/share/nginx/html# cat index.html 
Hello from Kubernetes storage
root@task-pv-pod:/usr/share/nginx/html# 

[root@master pv]# curl 10.244.1.163
Hello from Kubernetes storage
[root@master pv]# 
```

## 六、模拟网站更新效果

```shell
[root@node1 data]# cat index.html 
Hello from Kubernetes storage
hello xiaowei
[root@node1 data]# 
[root@master pv]# curl 10.244.1.163
Hello from Kubernetes storage
hello xiaowei
[root@master pv]# 
```

