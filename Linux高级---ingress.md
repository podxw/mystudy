# Linux高级---ingress

[TOC]

## 一、ingress介绍

![image-20230514134230024](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---ingress.assets%5Cimage-20230514134230024.png)

```markdown
    在前面课程中已经提到，Service对集群之外暴露服务的主要方式有两种：NotePort和LoadBalancer，但是这两种方式，都有一定的缺点：
```

- NodePort方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB方式的缺点是每个service需要一个LB，浪费、麻烦，并且需要kubernetes之外设备的支持

```markdown
    基于这种现状，kubernetes提供了Ingress资源对象，Ingress只需要一个NodePort或者一个LB就可以满足暴露多个Service的需求。工作机制大致如下图表示：
```

![image-20230513191417042](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---ingress.assets%5Cimage-20230513191417042.png)

实际上，Ingress相当于一个7层的负载均衡器，是kubernetes对反向代理的一个抽象，它的工作原理类似于Nginx，可以理解成在**Ingress里建立诸多映射规则，Ingress Controller通过监听这些配置规则并转化成Nginx的反向代理配置 , 然后对外部提供服务**。在这里有两个核心概念：

- ingress：kubernetes中的一个对象，作用是定义请求如何转发到service的规则
- ingress controller：具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发，实现方式有很多，比如Nginx, Contour, Haproxy等等

## 二、ingress的工作原理

>Ingress（以Nginx为例）的工作原理如下：

1. 用户编写Ingress规则，说明哪个域名对应kubernetes集群中的哪个Service
2. Ingress控制器动态感知Ingress服务规则的变化，然后生成一段对应的Nginx反向代理配置
3. Ingress控制器会将生成的Nginx配置写入到一个运行着的Nginx服务中，并动态更新
4. 到此为止，其实真正在工作的就是一个Nginx了，内部配置了用户定义的请求转发规则

![image-20230514133802108](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---ingress.assets%5Cimage-20230514133802108.png)

## 三、ingress的使用

### 1、搭建ingress环境	

```shell
[root@master ingress]# ls
ingress-controller-deploy.yaml  ingress-nginx-controllerv1.1.0.tar.gz  kube-webhook-certgen-v1.1.0.tar.gz  tomcat-nginx.yaml
[root@master ingress]# 
# ingress-controller-deploy.yaml 为安装ingress-nginx的yaml文件
# ingress-nginx-controllerv1.1.0.tar.gz 为安装ingress-ngix的镜像文件

# 执行安装完成之后

# 查看ingress-nginx
[root@master ingress]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS      AGE
ingress-nginx-controller-7cd558c647-6mvc5   1/1     Running     1 (13m ago)   133m
ingress-nginx-controller-7cd558c647-zcqq9   1/1     Running     1 (14m ago)   133m

# 查看service
[root@master ingress]# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.1.87.167   <none>        80:31047/TCP,443:30526/TCP   133m
ingress-nginx-controller-admission   ClusterIP   10.1.39.127   <none>        443/TCP                      133m
[root@master ingress]# 

```

### 2、准备service和pod

> 为了后面的实验比较方便，创建如下图所示的模型：

![image-20230514193642445](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---ingress.assets%5Cimage-20230514193642445.png)

>创建tomcat-nginx.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:latest
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```shell
# 创建pod
[root@master ingress]# kubectl apply -f tomcat-nginx.yaml 

# 查看相应的pod和service
[root@master ingress]# kubectl get pod -n dev
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6756f95949-dzsk8   1/1     Running   0          14m
nginx-deployment-6756f95949-jr7nq   1/1     Running   0          14m
nginx-deployment-6756f95949-kwhf9   1/1     Running   0          14m
tomcat-deployment-b5d5d98cb-hmplf   1/1     Running   0          14m
tomcat-deployment-b5d5d98cb-lftm9   1/1     Running   0          14m
tomcat-deployment-b5d5d98cb-x5fld   1/1     Running   0          14m
[root@master ingress]# kubectl get service -n dev
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
nginx-service    ClusterIP   None         <none>        80/TCP     14m
tomcat-service   ClusterIP   None         <none>        8080/TCP   14m
[root@master ingress]# 
```

### 3、创建http代理

>创建ingress-http.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
  - host: nginx.yuanke.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: tomcat.yuanke.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

```shell
# 创建
[root@master ingress]# kubectl create -f ingress-http.yaml  --validate=false
ingress.networking.k8s.io/ingress-http created

# 查看
[root@master ingress]# kubectl get ingress -n dev
NAME           CLASS    HOSTS                                ADDRESS   PORTS   AGE
ingress-http   <none>   nginx.yuanke.com,tomcat.yuanke.com             80      21s

# 查看详情
[root@master ingress]# kubectl describe ingress ingress-http -n dev
Name:             ingress-http
Labels:           <none>
Namespace:        dev
Address:          
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host               Path  Backends
  ----               ----  --------
  nginx.yuanke.com   
                     /   nginx-service:80 (10.244.1.96:80,10.244.1.97:80,10.244.2.99:80)
  tomcat.yuanke.com  
                     /   tomcat-service:8080 (10.244.1.96:80,10.244.1.97:80,10.244.2.99:80)
Annotations:         <none>
Events:              <none>
[root@master ingress]# 

```

http://nginx.yuanke.com:31047进行访问

### 4、创建https代理

>创建证书

```shell
# 生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=yuanke.com"

# 创建密钥
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

>创建ingress-https.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-https
  namespace: dev
spec:
  tls:
    - hosts:
      - nginx.yuanke.com
      - tomcat.yuanke.com
      secretName: tls-secret # 指定秘钥
  rules:
  - host: nginx.yuanke.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: tomcat.yuanke.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

```shell
# 创建
[root@master ingress]# kubectl apply -f ingress-https.yaml 
ingress.networking.k8s.io/ingress-https created

# 查看
[root@master ingress]# kubectl get ingress -n dev
NAME            CLASS    HOSTS                                ADDRESS   PORTS     AGE
ingress-http    <none>   nginx.yuanke.com,tomcat.yuanke.com             80, 443   15m
ingress-https   <none>   nginx.yuanke.com,tomcat.yuanke.com             80, 443   3s

# 查看详情
[root@master ingress]# kubectl describe ingress ingress-https -n dev
Name:             ingress-https
Labels:           <none>
Namespace:        dev
Address:          
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  tls-secret terminates nginx.yuanke.com,tomcat.yuanke.com
Rules:
  Host               Path  Backends
  ----               ----  --------
  nginx.yuanke.com   
                     /   nginx-service:80 (10.244.1.3:80,10.244.2.6:80,10.244.3.3:80)
  tomcat.yuanke.com  
                     /   tomcat-service:8080 (10.244.1.4:8080,10.244.2.5:8080,10.244.3.4:8080)
Annotations:         <none>
Events:              <none>
[root@master ingress]# 
```

https://nginx.yuanke.com:30526进行访问