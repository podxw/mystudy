# Linux高级---k8s三种探针readinessProbe、livenessProbe和startupProbe

[TOC]

## 一、POD状态

### 1、POD常见的状态

```
Pending：挂起，我们在请求创建pod时，条件不满足，调度没有完成，没有任何一个节点能满足调度条件。已经创建了但是没有适合它运行的节点叫做挂起，这其中也包含集群为容器创建网络，或者下载镜像的过程。    
Running：Pod内所有的容器都已经被创建，且至少一个容器正在处于运行状态、正在启动状态或者重启状态。    
Succeeded：Pod中所以容器都执行成功后退出，并且没有处于重启的容器。
Failed：Pod中所以容器都已退出，但是至少还有一个容器退出时为失败状态。
Unknown：未知状态，所谓pod是什么状态是apiserver和运行在pod节点的kubelet进行通信获取状态信息的，如果节点之上的kubelet本身出故障，那么apiserver就连不上kubelet，得不到信息了，就会看Unknown
```

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9yQTJRUVZHa2Q4UkFnV2ljbHFlSjE0UUpReW9IYWVBdzZGT1NpY01XaFV1bEQwSzF5VUVwbnJ0Q21pYjhlMGlhT0dQbWxZNkxHaWJzQ0xEMkxVd3NHMU43QVFRLzY0MA?x-oss-process=image/format,png)

### 2、POD重启策略

**Always:**   只要容器失效退出就重新启动容器。
**OnFailure:** 当容器以非正常(异常)退出后才自动重新启动容器。
**Never:**    无论容器状态如何，都不重新启动容器。

```
如果pod的restartpolicy没有设置，那么默认值是Always。
```

## 二、就绪、存活两种探针

**K8S 提供了3种探针**

- `readinessProbe`
- `livenessProbe`
- `startupProbe（这个1.16版本增加的）`

### 1、探针介绍

```
在 Kubernetes 中 Pod 是最小的计算单元，而一个 Pod 又由多个容器组成，相当于每个容器就是一个应用，应用在运行期间，可能因为某也意外情况致使程序挂掉。那么如何监控这些容器状态稳定性，保证服务在运行期间不会发生问题，发生问题后进行重启等机制，就成为了重中之重的事情，考虑到这点 kubernetes 推出了活性探针机制。有了存活性探针能保证程序在运行中如果挂掉能够自动重启，但是还有个经常遇到的问题，比如说，在Kubernetes 中启动Pod，显示明明Pod已经启动成功，且能访问里面的端口，但是却返回错误信息。还有就是在执行滚动更新时候，总会出现一段时间，Pod对外提供网络访问，但是访问却发生404，这两个原因，都是因为Pod已经成功启动，但是 Pod 的的容器中应用程序还在启动中导致，考虑到这点Kubernetes推出了就绪性探针机制。
```

### 2、livenessProbe

>**livenessProbe：存活性探针，用于判断容器是不是健康，如果不满足健康条件，那么 Kubelet 将根据 Pod 中设置的 restartPolicy （重启策略）来判断，Pod 是否要进行重启操作**。LivenessProbe按照配置去探测 ( 进程、或者端口、或者命令执行后是否成功等等)，来判断容器是不是正常。如果探测不到，代表容器不健康（可以配置连续多少次失败才记为不健康），则 kubelet 会杀掉该容器，并根据容器的重启策略做相应的处理。如果未配置存活探针，则默认容器启动为通过（Success）状态。即探针返回的值永远是 Success。即Success后pod状态是RUNING

### 3、readinessProbe

>**readinessProbe 就绪性探针，用于判断容器内的程序是否存活（或者说是否健康），只有程序(服务)正常， 容器开始对外提供网络访问（启动完成并就绪）**。容器启动后按照readinessProbe配置进行探测，无问题后结果为成功即状态为 Success。pod的READY状态为 true，从0/1变为1/1。如果失败继续为0/1，状态为 false。若未配置就绪探针，则默认状态容器启动后为Success。对于此pod、此pod关联的Service资源、EndPoint 的关系也将基于 Pod 的 Ready 状态进行设置，如果 Pod 运行过程中 Ready 状态变为 false，则系统自动从 Service资源 关联的 EndPoint 列表中去除此pod，届时service资源接收到GET请求后，kube-proxy将一定不会把流量引入此pod中，通过这种机制就能防止将流量转发到不可用的 Pod 上。如果 Pod 恢复为 Ready 状态。将再会被加回 Endpoint 列表。kube-proxy也将有概率通过负载机制会引入流量到此pod中。

### 4、就绪、存活两种探针的区别

```
ReadinessProbe 和 livenessProbe 可以使用相同探测方式，只是对 Pod 的处置方式不同：
readinessProbe 当检测失败后，将 Pod 的 IP:Port 从对应的 EndPoint 列表中删除。
livenessProbe 当检测失败后，将杀死容器并根据 Pod 的重启策略来决定作出对应的措施。
```

### 5、**就绪**、**存活**两种探针的使用方法

**目前 LivenessProbe 和 ReadinessProbe 两种探针都支持下面三种探测方法：**

```
ExecAction：在容器中执行指定的命令，如果执行成功，退出码为 0 则探测成功。
HTTPGetAction：通过容器的IP地址、端口号及路径调用 HTTP Get方法，如果响应的状态码大于等于200且小于400，则认为容器 健康。
TCPSocketAction：通过容器的 IP 地址和端口号执行 TCP 检 查，如果能够建立 TCP 连接，则表明容器健康。
 
探针探测结果有以下值：
Success：表示通过检测。
Failure：表示未通过检测。
Unknown：表示检测没有正常进行。
```

**LivenessProbe 和 ReadinessProbe 两种探针的相关属性**

```
探针(Probe)有许多可选字段，可以用来更加精确的控制Liveness和Readiness两种探针的行为(Probe)：
 
initialDelaySeconds：容器启动后要等待多少秒后就探针开始工作，单位“秒”，默认是 0 秒，最小值是 0
periodSeconds：执行探测的时间间隔（单位是秒），默认为 10s，单位“秒”，最小值是 1
timeoutSeconds：探针执行检测请求后，等待响应的超时时间，默认为 1s，单位“秒”，最小值是 1
successThreshold：探针检测失败后认为成功的最小连接成功次数，默认为 1s，在 Liveness 探针中必须为 1s，最小值为 1s。
failureThreshold：探测失败的重试次数，重试一定次数后将认为失败，在 readiness 探针中，Pod会被标记为未就绪，默认为 3s，最小值为 1s
 
注：initialDelaySeconds在readinessProbe其实可以不用配置，不配置默认pod刚启动，开始进行readinessProbe探测，但那有怎么样，除了
startupProbe，readinessProbe、livenessProbe运行在pod的整个生命周期，刚启动的时候readinessProbe检测失败了，只不过显示READY状态一直是0/1，readinessProbe失败并不会导致重启pod，只有startupProbe、livenessProbe失败才会重启pod。而等到多少s后，真正服务启动后，检查success成功后，READY状态自然正常
```

## 三、LivenessProbe探针

### 1、通过exec方式做健康探测

```shell
[root@master LivenessProbe]# cat liveness-exec.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  labels:
    app: liveness
spec:
  containers:
  - name: liveness
    image: busybox
    args:                       #创建测试探针探测的文件
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      initialDelaySeconds: 1    #延迟检测时间
      periodSeconds: 5          #检测时间间隔
      exec:                     #使用命令检查
        command:                #指令，类似于运行命令sh
        - cat                   #sh 后的第一个内容，直到需要输入空格，变成下一行
        - /tmp/healthy          #由于不能输入空格，需要另外声明，结果为sh cat"空格"/tmp/healthy
[root@master LivenessProbe]# 
```

>解释整体意思：
>
>容器在初始化后，执行（/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 60"）首先创建一个 /tmp/healthy 文件，然后执行睡眠命令，睡眠 30 秒，到时间后执行删除 /tmp/healthy 文件命令。而设置的存活探针检检测方式为执行 shell 命令，用 cat 命令输出 healthy 文件的内容，如果能成功执行这条命令一次(默认successThreshold:1)，存活探针就认为探测成功，由于没有配置(failureThreshold、timeoutSeconds)，所以执行（cat /tmp/healthy）并只等待1s，如果1s内执行后返回失败，探测失败。在前 30 秒内，由于文件存在，所以存活探针探测时执行 cat /tmp/healthy 命令成功执行。30 秒后 healthy 文件被删除，所以执行命令失败，Kubernetes 会根据 Pod 设置的重启策略来判断，是否重启 Pod。

```shell
[root@master LivenessProbe]# kubectl get pod -w
NAME                          READY   STATUS    RESTARTS   AGE
liveness-exec                 1/1     Running   0          18s
liveness-exec                 1/1     Running   1 (41s ago)   2m6s
liveness-exec                 1/1     Running   2 (0s ago)    2m40s
# 三十秒之后就会重启pod
```

### 2、通过HTTP方式做健康探测

 **httpGet探测方式有如下可选的控制字段:**

```
scheme: 用于连接host的协议，默认为HTTP。
host：要连接的主机名，默认为Pod IP，可以在http request head中设置host头部。
port：容器上要访问端口号或名称。
path：http服务器上的访问URI。
httpHeaders：自定义HTTP请求headers，HTTP允许重复headers。
```

```shell
[root@master LivenessProbe]# cat liveness-http.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
  labels:
    test: liveness
spec:
  containers:
  - name: liveness
    image: nginx
    imagePullPolicy: IfNotPresent
    livenessProbe:
      initialDelaySeconds: 1   #延迟加载时间
      periodSeconds: 10          #重试时间间隔
      timeoutSeconds: 5         #超时时间设置
      httpGet:
        scheme: HTTP
        port: 80
        path: /index.html
[root@master LivenessProbe]# kubectl apply -f liveness-http.yaml 
pod/liveness-http created
[root@master LivenessProbe]# kubectl get pod -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
liveness-http   1/1     Running   0          4s    10.244.2.235   node2   <none>           <none>
[root@master LivenessProbe]# curl 10.244.2.235
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
[root@master LivenessProbe]# 
```

```shell
# 容器能够启动，能够访问到index.html首页文件，现在我们进入容器将首页删掉。这样可以看到容器被重启
[root@master LivenessProbe]# kubectl exec -it liveness-http -- rm -rf /usr/share/nginx/html/index.html
[root@master LivenessProbe]# kubectl get pod -o wide
NAME            READY   STATUS    RESTARTS     AGE     IP             NODE    NOMINATED NODE   READINESS GATES
liveness-http   1/1     Running   1 (1s ago)   3m21s   10.244.2.235   node2   <none>           <none>
[root@master LivenessProbe]# 
```

### 3、通过TCP方式做健康探测

```shell
[root@master LivenessProbe]# cat liveness-tcp.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
  labels:
    app: liveness-tcp
spec:
  containers:
  - name: liveness-tcp
    image: nginx
    imagePullPolicy: IfNotPresent
    livenessProbe:
      initialDelaySeconds: 5
      periodSeconds: 3
      timeoutSeconds: 1
      tcpSocket:
        port: 8080
[root@master LivenessProbe]# 
# 首先是启动容器，五秒后检查容器的8080端口，发现没有这个端口就会重启
```

```shell
[root@master LivenessProbe]# kubectl apply -f liveness-tcp.yaml 
pod/liveness-tcp created
[root@master LivenessProbe]# kubectl get pod -w
NAME           READY   STATUS    RESTARTS   AGE
liveness-tcp   1/1     Running   0          7s
liveness-tcp   1/1     Running   1 (1s ago)   13s
liveness-tcp   1/1     Running   2 (1s ago)   25s
# 可以发现检查五秒后就会检查失败，就会重启，再检查再重启
```

## 四、ReadnessProbe探针

>Pod 的ReadinessProbe 探针使用方式和 LivenessProbe 探针探测方法一样，也是支持三种
>
>就绪探测。探测失败，端点控制器将从与pod匹配的所有svc端点中删除pod的ip地址

### 1、通过HTTP方式做健康探测

```shell
[root@master ReadnessProbe]# cat readness-http.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget-pod
  namespace: default
spec:
  containers:
  - name: readiness-httpget-container
    image: nginx
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
      initialDelaySeconds: 1
      periodSeconds: 3
[root@master ReadnessProbe]# 
# 该探针一直查找index.html的首页文件，一直找不到，就会一直起不来，不会处于ready状态
[root@master ReadnessProbe]# kubectl apply -f readness-http.yaml 
pod/readiness-httpget-pod created
[root@master ReadnessProbe]# curl 10.244.2.237/index1.html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.23.3</center>
</body>
</html>
[root@master ReadnessProbe]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   0/1     Running   0          5s
[root@master ReadnessProbe]# 
```

```shell
# 我们进入容器新建一个index1.html文件即可启动起来。
[root@master ReadnessProbe]# kubectl exec -it readiness-httpget-pod -- bash
root@readiness-httpget-pod:/# cd /usr/share/nginx/html/
root@readiness-httpget-pod:/usr/share/nginx/html# echo "123" > index1.html 
root@readiness-httpget-pod:/usr/share/nginx/html# 
exit
[root@master ReadnessProbe]# kubectl get pod 
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   1/1     Running   0          4m18s
[root@master ReadnessProbe]# 
```

### 2、liveness和readness结合

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget-pod
  namespace: default
spec:
  containers:
  - name: readiness-httpget-container
    image: nginx
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
      initialDelaySeconds: 1
      periodSeconds: 3
      timeoutSeconds: 1          #超时时间设置
    livenessProbe:
      initialDelaySeconds: 1     #延迟加载时间
      periodSeconds: 3           #重试时间间隔
      timeoutSeconds: 1          #超时时间设置
      httpGet:
        scheme: HTTP
        port: 80
        path: /index.html
```

```
上面这个是liveness和readness的结合，在容器启动的一秒后分别检查，readness检查是否有index1.html文件，若没有则不ready，liveness检查是否有index.html文件，若没有则重启容器。
```

```shell
[root@master read-live]# kubectl apply -f read-live.yaml 
pod/readiness-httpget-pod created
[root@master read-live]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   0/1     Running   0          6s
[root@master read-live]# 
# 一直处于不起来的状态，当我们新建一个index1.html 文件时，这个容器就能起来了。
[root@master read-live]# kubectl exec -it readiness-httpget-pod -- bash
root@readiness-httpget-pod:/# cd /usr/share/nginx/html/
root@readiness-httpget-pod:/usr/share/nginx/html# echo "123" > index1.html 
root@readiness-httpget-pod:/usr/share/nginx/html# 
exit
[root@master read-live]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   1/1     Running   0          3m1s
[root@master read-live]# 
```

## 五、startupProbe探针

```shell
指检测容器中的应用是否已经启动。

如果提供了启动探测(startup probe)，则禁用所有其他探测，直到它成功为止。
如果启动探测失败，kubelet 将杀死容器，容器服从其重启策略进行重启。
如果容器没有提供启动探测，则默认状态为成功Success。
主要解决在慢启动程序或复杂程序中readinessProbe、livenessProbe探针无法较好的判断程序是否启动、是否存活。

引入startupProbe探针是为readinessProbe、livenessProbe探针服务。
```

### 1、执行顺序

```
如果三个探针同时存在，则先执行startupProbe探针，其他两个探针将会被暂时禁用，直到startupProbe一次探测成功，其他2个探针才启动；如果startupProbe探测失败，kubelet 将杀死容器，并根据restartPolicy重启策略来判断容器是否要进行重启操作。

就绪探针与存活探针之间的重要区别：如果容器未通过准备检查，则不会被终止或重新启动。
存活探针：通过杀死异常的容器，并用新的正常容器替代他们来保持Pod正常工作
就绪探针：确保只有准备好处理请求的Pod才可以接收探针请求
```

