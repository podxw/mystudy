# linux高级---k8s 使用 Service 控制器对外暴露服务

>Service 引入主要是解决 Pod 的动态变化，提供统一访问入口：
>
>1. 防止 Pod 失联，准备找到提供同一个服务的 Pod （服务发现） 
>2. 定义一组 Pod 的访问策略 （负载均衡）

## 一、什么是服务service？

在k8s里面，每个Pod都会被分配一个单独的IP地址,但这个IP地址会随着Pod的销毁而消失，重启pod的ip地址会发生变化，此时客户如果访问原先的ip地址则会报错 ；

>Service (服务)就是用来解决这个问题的, 对外服务的统一入口，防止pod失联，定义一组pod的访问策略（服务发现、负载均衡） ；

一个Service可以看作一组提供相同服务的Pod的对外访问接口，作用于哪些Pod是通过标签选择器来定义的 ，Service是一个概念，主要作用的是节点上的kube-proxy服务进程 ；

举例来说，定义了3个商品微服务，由网关作为统一访问入口， 前端不需要关心它们调用了哪个后端副本。 然而组成这一组后端程序的 Pod 实际上可能会发生变化， 前端客户端不应该也没必要知道，而且也不需要跟踪这一组后端的状态。

>简而言之，Service 定义的这种抽象能够解耦这种关联；

service的作用：

![image-20230313211657060](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230313211657060.png)

## 二、service分类

根据使用场景的不同，service可以分成下面几类

```
1 ClusterIP
默认类型，自动分配一个【仅集群内部】可以访问的虚拟IP

2 NodePort
对外访问应用使用，在ClusterIP基础上为Service在每台机器上绑定一个端口，就可以通过: ip+NodePort来访问该服务 ，在之前搭建k8s集群部署nginx的时候我们使用过；

3 LoadBalancer（付费方案）
使在NodePort的基础上，借助公有云创建一个外部负载均衡器，并将请求转发到NodePort ；
可以实现集群外部访问服务的另外一种解决方案，不过并不是所有的k8s集群都会支持，大多是在公有云托管集群中会支持该类型 ；

4 ExternalName （使用较少）
把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何类型代理被创建，这只有 Kubernetes 1.7或更高版本的kube-dns才支持。
```

## 三、service和pod的关系

service和pod之间是通过 selector.app进行关联的 ，对应到yaml中的核心配置如下：

```yaml
spec: # 描述
  selector: # 标签选择器，确定当前service代理控制哪些pod
    app: test-nginx
```

## 四、Service 之 ClusterIP 使用

ClusterIP 属于service的一种，一般作为集群内部应用互相访问时使用，接下来通过实际演示进行详细的说明；

### 1、创建一个deploy-nginx-pod.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xw-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: xw-nginx-pod
  template:
    metadata:
      labels:
        app: xw-nginx-pod
    spec:
      containers:
      - name: xw-nginx
        image: nginx
```

```shell
# 可以看到，成功创建了一个3副本的pod集群
[root@master clusterip]# kubectl apply -f deploy-nginx-podyaml 
deployment.apps/xw-deploy created
[root@master clusterip]# kubectl get pod
NAME                          READY   STATUS    RESTARTS      AGE
xw-deploy-7cf897fb4d-7hf9b    1/1     Running   0             13s
xw-deploy-7cf897fb4d-8lwj6    1/1     Running   0             13s
xw-deploy-7cf897fb4d-92fwj    1/1     Running   0             13s
[root@master clusterip]# kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
xw-deploy    3/3     3            3           27s
[root@master clusterip]# 
```

### 2、暴露服务为 clusterIP 类型

```shell
[root@master clusterip]# kubectl expose deploy xw-deploy --name=svc-nginx-1 --type=ClusterIP --port=80 --target-port=80 -n default
service/svc-nginx-1 exposed
[root@master clusterip]# kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
svc-nginx-1   ClusterIP   10.1.173.192   <none>        80/TCP    5s
```

### 3、查看服务

```shell
# 通过curl clusterIp+port可以访问
[root@master clusterip]# curl 10.1.173.192:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@master clusterip]# 
```

## 五、Service 之 NodePort 使用

上面了解到clusterIp这种方式一般作为集群内部各应用访问时使用，但实际业务场景中，有些应用服务需要暴露出去，通过外网去访问，这时候就需要创建NodePort

### 1、关于NodePort

```
常规业务的场景不全是集群内访问，也需要集群外业务访问 ；
那么ClusterIP就满足不了了，NodePort是其中的一种实现集群外部访问的方案 ；
对外访问应用使用，在ClusterIP基础上为Service在每台机器上绑定一个端口，就可以通过: ip+NodePort来访问该服务 ；
```

### 2、创建NodePort类型的服务

使用下面的命令创建一个基于 NodePort 的service

```shell
[root@master clusterip]# kubectl expose deploy xw-deploy --name=svc-nginx-2 --type=NodePort --port=80 --target-port=80 -n default
service/svc-nginx-2 exposed
[root@master clusterip]# kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
svc-nginx-1   ClusterIP   10.1.173.192   <none>        80/TCP         3m48s
svc-nginx-2   NodePort    10.1.90.35     <none>        80:30675/TCP   3s
[root@master clusterip]# kubectl describe svc svc-nginx-2
Name:                     svc-nginx-2
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=xw-nginx-pod
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.1.90.35
IPs:                      10.1.90.35
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30675/TCP
Endpoints:                10.244.1.209:80,10.244.1.210:80,10.244.2.225:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
[root@master clusterip]# 
```

使用浏览器访问下当前主机的公网IP就可以访问了（master或者node节点都可以访问）；

![image-20230313211241124](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230313211241124.png)

### 3、关于NodePort暴露对外端口说明

```
Kubeadm部署，暴露端口对外服务，会随机选端口，默认范围是30000~32767，也可以手动修改和指定端口范围
```

### 4、关于Endpoint参数说明

```
是k8s中的一个资源对象，存储在etcd中，记录service对应的所有pod的访问地址 ；
里面有个Endpoints列表，就是当前service可以负载到的pod服务入口 ；
service和pod之间的通信是通过endpoint实现的 ；
```

可以使用下面的命令查看endpoint列表

```shell
[root@master clusterip]# kubectl describe svc svc-nginx-2
Name:                     svc-nginx-2
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=xw-nginx-pod
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.1.90.35
IPs:                      10.1.90.35
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30675/TCP
Endpoints:                10.244.1.209:80,10.244.1.210:80,10.244.2.225:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
[root@master clusterip]# 
```

### 5、关于Service如何分发请求到后端的多个Pod

Service所处的位置就如同图中展示的，处在客户端和pod之间，这就很像nginx或者其他具备网关的组件的能力了，事实上，也差不多就是我们理解的那样，一个Service的服务背后可能挂载着N多个Pod，那么具体来说，Service如何分发请求到后端的多个Pod的呢？

![image-20230313211458678](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230313211458678.png)

```
kubernetes提供了两种负载均衡策略 ：
默认，kube-proxy的策略，如随机、轮询 ；
使用会话保持模式，即同个客户端的请求固定到某个pod，在spec中添加sessionAffinity:ClientIP即可 ；
```