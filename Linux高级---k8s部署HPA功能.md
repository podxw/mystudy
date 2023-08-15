# Linux高级---k8s部署HPA功能

[TOC]

## 一、确保metrics server安装好

```shell
[root@master metrics]# kubectl get pod -n kube-system -o wide| grep metrics
metrics-server-b9f7b695f-dx9qf   1/1     Running   0             22m     10.244.1.155     node1    <none>           <none>
[root@master metrics]# 
```

## 二、创建hpa准备工作

>官方文档：https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

### 1、编辑Dockerfile和index.php

```shell
[root@master hpa]# cat Dockerfile 
FROM php:5-apache
COPY index.php /var/www/html/index.php
RUN chmod a+rx index.php
[root@master hpa]# cat index.php 
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
[root@master hpa]# 
```

### 2、生成镜像文件

```shell
[root@master hpa]# docker build -t sc-hpa:1.0 .
[root@master hpa]# docker images
REPOSITORY                                                        TAG        IMAGE ID       CREATED         SIZE
sc-hpa                                                            1.0        ad9d172cbac0   29 hours ago    355MB
```

### 3、编写php-apache.yaml文件,并执行

```shell
[root@master hpa]# cat php-apache.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: sc-hpa:1.0
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 200m
          requests:
            cpu: 100m

---

apiVersion: v1
kind: Service
metadata:
  namespace: default
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

```shell
[root@master hpa]# kubectl apply -f php-apache.yaml 
deployment.apps/php-apache created
service/php-apache created
[root@master hpa]# 
[root@master hpa]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
php-apache-7c97954b84-d9jng   1/1     Running   0          59s   10.244.2.202   node2   <none>           <none>
[root@master hpa]# kubectl get rs
NAME                    DESIRED   CURRENT   READY   AGE
php-apache-7c97954b84   1         1         1       60s
[root@master hpa]# 
```

### 4、将sc-hap镜像导出，导入其他的node节点上

```shell
[root@master hpa]# docker save > sc-hap.tar sc-hpa:1.0 
[root@master hpa]# ls
Dockerfile  index.php  php-apache.yaml  sc-hap.tar
[root@master hpa]# scp sc-hap.tar 192.168.17.148:/root # node1节点
[root@master hpa]# scp sc-hap.tar 192.168.17.149:/root # node2节点
```

```shell
# node1节点
[root@node1 ~]# ls
anaconda-ks.cfg  sc-hpa.tar
[root@node1 ~]# docker load sc-hpa.tar 
[root@node1 ~]# docker images
REPOSITORY                                               TAG        IMAGE ID       CREATED         SIZE
sc-hpa                                                   1.0        ad9d172cbac0   29 hours ago    355MB
```

>node2同理一样的操作

## 三、创建hpa功能

### 1、创建hpa

```shell
[root@master hpa]# kubectl autoscale deployment php-apache --cpu-percent=20 --min=1 --max=10
[root@master hpa]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/20%    1         10        1          28s
[root@master hpa]# 
```

### 2、测试pod，增加它的负载

```shell
[root@master hpa]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
php-apache-7c97954b84-d9jng   1/1     Running   0          11m   10.244.2.202   node2   <none>           <none>
[root@master hpa]# curl 10.244.2.202
OK![root@master hpa]# 
```

```shell
# 压力测试
[root@master hpa]# kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; doet -q -O- http://10.244.2.202; done"
If you don't see a command prompt, try pressing enter.
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!^Cpod "load-generator" deleted
pod default/load-generator terminated (Error)
[root@master hpa]# 
[root@master hpa]# kubectl get hpa php-apache --watch
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/20%    1         10        1          29h
php-apache   Deployment/php-apache   138%/20%   1         10        1          29h
php-apache   Deployment/php-apache   201%/20%   1         10        4          29h
php-apache   Deployment/php-apache   200%/20%   1         10        8          29h
php-apache   Deployment/php-apache   51%/20%    1         10        10         29h
------

[root@master hpa]# kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
load-generator                1/1     Running   0          72s
php-apache-7c97954b84-c25xz   1/1     Running   0          30s
php-apache-7c97954b84-d9jng   1/1     Running   0          14m
php-apache-7c97954b84-gzj5j   1/1     Running   0          15s
php-apache-7c97954b84-lw57f   1/1     Running   0          45s
php-apache-7c97954b84-lwr29   1/1     Running   0          15s
php-apache-7c97954b84-rbhrg   1/1     Running   0          30s
php-apache-7c97954b84-rbk86   1/1     Running   0          45s
php-apache-7c97954b84-rdcqf   1/1     Running   0          30s
php-apache-7c97954b84-tjdjq   1/1     Running   0          30s
php-apache-7c97954b84-w5xd8   1/1     Running   0          45s
[root@master hpa]# 
```

```shell
# 停止压力测试，pod数又会减少
[root@master hpa]# kubectl get hpa php-apache --watch
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/20%    1         10        1          29h
php-apache   Deployment/php-apache   138%/20%   1         10        1          29h
php-apache   Deployment/php-apache   201%/20%   1         10        4          29h
php-apache   Deployment/php-apache   200%/20%   1         10        8          29h
php-apache   Deployment/php-apache   51%/20%    1         10        10         29h
php-apache   Deployment/php-apache   23%/20%    1         10        10         29h
php-apache   Deployment/php-apache   21%/20%    1         10        10         29h
php-apache   Deployment/php-apache   20%/20%    1         10        10         29h
php-apache   Deployment/php-apache   21%/20%    1         10        10         29h
php-apache   Deployment/php-apache   16%/20%    1         10        10         29h
php-apache   Deployment/php-apache   1%/20%     1         10        10         29h
php-apache   Deployment/php-apache   1%/20%     1         10        10         29h
php-apache   Deployment/php-apache   1%/20%     1         10        8          29h
php-apache   Deployment/php-apache   1%/20%     1         10        1          29h


[root@master hpa]# kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
php-apache-7c97954b84-d9jng   1/1     Running   0          22m
[root@master hpa]# 

```

