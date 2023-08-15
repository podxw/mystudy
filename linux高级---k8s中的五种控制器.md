# linux高级---k8s中的五种控制器

[TOC]

## 一、k8s的控制器类型

Kubernetes中内建了很多controller（控制器），这些相当于一个状态机，用来控制Pod的具体状态和行为

```
Deployment：适合无状态的服务部署
StatefullSet：适合有状态的服务部署
DaemonSet：一次部署，所有的node节点都会部署，例如一些典型的应用场景：
运行集群存储 daemon，例如在每个Node上运行 glusterd、ceph
在每个Node上运行日志收集 daemon，例如 fluentd、 logstash
在每个Node上运行监控 daemon，例如 Prometheus Node Exporter
Job：一次性的执行任务
Cronjob：周期性的执行任务
```

总体来说，K8S有五种控制器，分别对应处理无状态应用、有状态应用、守护型应用和批处理应用

## 二、pod与控制器之间的关系

```
controllers：在集群上管理和运行容器的对象通过label-selector相关联
Pod通过控制器实现应用的运维，如伸缩，升级等
```

![image-20230313192819908](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230313192819908.png)

## 三、状态与无状态化对特点

>**无状态服务的特点：**

```
1）deployment 认为所有的pod都是一样的
2）不用考虑顺序的要求
3）不用考虑在哪个node节点上运行
4）可以随意扩容和缩容
```

>**有状态服务的特点：**

```
1）实例之间有差别，每个实例都有自己的独特性，元数据不同，例如etcd，zookeeper
2）实例之间不对等的关系，以及依靠外部存储的应用。
```

## 四、Deployment

```
1、Deployment:一般用来部署长期运行的、无状态的应用
特点：集群之中，随机部署

通过Deployment对象，你可以轻松的做到以下事情：
创建ReplicaSet和Pod
滚动升级（不停止旧服务的状态下升级）和回滚应用（将应用回滚到之前的版本）
平滑地扩容和缩容
暂停和继续Deployment
```

Deployment主要功能有下面几个：

- 支持ReplicaSet的所有功能
- 支持发布的停止、继续
- 支持滚动升级和回滚版本

### 1、Deployment的资源清单文件

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留历史版本
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

### 2、在配置清单中调用deployment控制器

创建pc-deployment.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
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
```

```powershell
# 创建deployment
[root@master ~]# kubectl create -f pc-deployment.yaml --record=true
deployment.apps/pc-deployment created

# 查看deployment
# UP-TO-DATE 最新版本的pod的数量
# AVAILABLE  当前可用的pod的数量
[root@master ~]# kubectl get deploy pc-deployment 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   3/3     3            3           15s

# 查看rs
# 发现rs的名称是在原来deployment的名字后面添加了一个10位数的随机串
[root@master ~]# kubectl get rs 
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   3         3         3       23s

# 查看pod
[root@master ~]# kubectl get pods 
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          107s
pc-deployment-6696798b78-smpvp   1/1     Running   0          107s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          107s
```

扩缩容

```powershell
# 变更副本数量为5个
[root@master ~]# kubectl scale deploy pc-deployment --replicas=5  
deployment.apps/pc-deployment scaled

# 查看deployment
[root@master ~]# kubectl get deploy pc-deployment 
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   5/5     5            5           2m

# 查看pod
[root@master ~]#  kubectl get pods 
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          4m19s
pc-deployment-6696798b78-jxmdq   1/1     Running   0          94s
pc-deployment-6696798b78-mktqv   1/1     Running   0          93s
pc-deployment-6696798b78-smpvp   1/1     Running   0          4m19s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          4m19s

# 编辑deployment的副本数量，修改spec:replicas: 4即可
[root@master ~]# kubectl edit deploy pc-deployment  
deployment.apps/pc-deployment edited

# 查看pod
[root@master ~]# kubectl get pods 
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6696798b78-d2c8n   1/1     Running   0          5m23s
pc-deployment-6696798b78-jxmdq   1/1     Running   0          2m38s
pc-deployment-6696798b78-smpvp   1/1     Running   0          5m23s
pc-deployment-6696798b78-wvjd8   1/1     Running   0          5m23s
```

### 3、镜像更新

deployment支持两种更新策略:`重建更新`和`滚动更新`,可以通过`strategy`指定策略类型,支持两个属性:

```markdown
strategy：指定新的Pod替换旧的Pod的策略， 支持两个属性：
  type：指定策略类型，支持两种策略
    Recreate：在创建出新的Pod之前会先杀掉所有已存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本Pod
  rollingUpdate：当type为RollingUpdate时生效，用于为RollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

重建更新

1) 编辑pc-deployment.yaml,在spec节点下添加更新策略

```yaml
spec:
  strategy: # 策略
    type: Recreate # 重建更新
```

2) 创建deploy进行验证

```powershell
# 变更镜像
[root@master ~]# kubectl set image deployment pc-deployment nginx=nginx:1.17.2 
deployment.apps/pc-deployment image updated

# 观察升级过程
[root@master ~]#  kubectl get pods  -w
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-65qcw   1/1     Running   0          31s
pc-deployment-5d89bdfbf9-w5nzv   1/1     Running   0          31s
pc-deployment-5d89bdfbf9-xpt7w   1/1     Running   0          31s

pc-deployment-5d89bdfbf9-xpt7w   1/1     Terminating   0          41s
pc-deployment-5d89bdfbf9-65qcw   1/1     Terminating   0          41s
pc-deployment-5d89bdfbf9-w5nzv   1/1     Terminating   0          41s

pc-deployment-675d469f8b-grn8z   0/1     Pending       0          0s
pc-deployment-675d469f8b-hbl4v   0/1     Pending       0          0s
pc-deployment-675d469f8b-67nz2   0/1     Pending       0          0s

pc-deployment-675d469f8b-grn8z   0/1     ContainerCreating   0          0s
pc-deployment-675d469f8b-hbl4v   0/1     ContainerCreating   0          0s
pc-deployment-675d469f8b-67nz2   0/1     ContainerCreating   0          0s

pc-deployment-675d469f8b-grn8z   1/1     Running             0          1s
pc-deployment-675d469f8b-67nz2   1/1     Running             0          1s
pc-deployment-675d469f8b-hbl4v   1/1     Running             0          2s
```

滚动更新

1) 编辑pc-deployment.yaml,在spec节点下添加更新策略

```yaml
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate:
      maxSurge: 25% 
      maxUnavailable: 25%
```

2) 创建deploy进行验证

```powershell
# 变更镜像
[root@master ~]# kubectl set image deployment pc-deployment nginx=nginx:1.17.3 
deployment.apps/pc-deployment image updated

# 观察升级过程
[root@master ~]# kubectl get pods -n dev -w
NAME                           READY   STATUS    RESTARTS   AGE
pc-deployment-c848d767-8rbzt   1/1     Running   0          31m
pc-deployment-c848d767-h4p68   1/1     Running   0          31m
pc-deployment-c848d767-hlmz4   1/1     Running   0          31m
pc-deployment-c848d767-rrqcn   1/1     Running   0          31m

pc-deployment-966bf7f44-226rx   0/1     Pending             0          0s
pc-deployment-966bf7f44-226rx   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-226rx   1/1     Running             0          1s
pc-deployment-c848d767-h4p68    0/1     Terminating         0          34m

pc-deployment-966bf7f44-cnd44   0/1     Pending             0          0s
pc-deployment-966bf7f44-cnd44   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-cnd44   1/1     Running             0          2s
pc-deployment-c848d767-hlmz4    0/1     Terminating         0          34m

pc-deployment-966bf7f44-px48p   0/1     Pending             0          0s
pc-deployment-966bf7f44-px48p   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-px48p   1/1     Running             0          0s
pc-deployment-c848d767-8rbzt    0/1     Terminating         0          34m

pc-deployment-966bf7f44-dkmqp   0/1     Pending             0          0s
pc-deployment-966bf7f44-dkmqp   0/1     ContainerCreating   0          0s
pc-deployment-966bf7f44-dkmqp   1/1     Running             0          2s
pc-deployment-c848d767-rrqcn    0/1     Terminating         0          34m

# 至此，新版本的pod创建完毕，就版本的pod销毁完毕
# 中间过程是滚动进行的，也就是边销毁边创建
```

 更新时候会创建新的replicaset。

**版本回退**

deployment支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能，下面具体来看.

kubectl rollout： 版本升级相关功能，支持下面的选项：

- status       显示当前升级状态
- history     显示 升级历史记录
- pause       暂停版本升级过程
- resume    继续已经暂停的版本升级过程
- restart      重启版本升级过程
- undo        回滚到上一级版本（可以使用--to-revision回滚到指定版本）

```powershell
# 查看当前升级版本的状态
[root@master ~]# kubectl rollout status deploy pc-deployment 
deployment "pc-deployment" successfully rolled out

# 查看升级历史记录
[root@master ~]# kubectl rollout history deploy pc-deployment 
deployment.apps/pc-deployment
REVISION  CHANGE-CAUSE
1         kubectl create --filename=pc-deployment.yaml --record=true
2         kubectl create --filename=pc-deployment.yaml --record=true
3         kubectl create --filename=pc-deployment.yaml --record=true
# 可以发现有三次版本记录，说明完成过两次升级

# 版本回滚
# 这里直接使用--to-revision=1回滚到了1版本， 如果省略这个选项，就是回退到上个版本，就是2版本
[root@master ~]# kubectl rollout undo deployment pc-deployment --to-revision=1 
deployment.apps/pc-deployment rolled back

# 查看发现，通过nginx镜像版本可以发现到了第一版
[root@master ~]# kubectl get deploy  -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         
pc-deployment   4/4     4            4           74m   nginx        nginx:1.17.1   

# 查看rs，发现第一个rs中有4个pod运行，后面两个版本的rs中pod为运行
# 其实deployment之所以可是实现版本的回滚，就是通过记录下历史rs来实现的，
# 一旦想回滚到哪个版本，只需要将当前版本pod数量降为0，然后将回滚版本的pod提升为目标数量就可以了
[root@master ~]# kubectl get rs
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   4         4         4       78m
pc-deployment-966bf7f44    0         0         0       37m
pc-deployment-c848d767     0         0         0       71m
```

### 4、金丝雀发布

​    Deployment控制器支持控制更新过程中的控制，如“暂停(pause)”或“继续(resume)”更新操作。

​    比如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

```powershell
# 更新deployment的版本，并配置暂停deployment
[root@master ~]#  kubectl set image deploy pc-deployment nginx=nginx:1.17.4  && kubectl rollout pause deployment pc-deployment  -n dev
deployment.apps/pc-deployment image updated
deployment.apps/pc-deployment paused

#观察更新状态
[root@master ~]# kubectl rollout status deploy pc-deployment 
Waiting for deployment "pc-deployment" rollout to finish: 2 out of 4 new replicas have been updated...

# 监控更新的过程，可以看到已经新增了一个资源，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了pause暂停命令

[root@master ~]# kubectl get rs  -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   3         3         3       19m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       14m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   2         2         2       3m16s   nginx        nginx:1.17.4   
[root@master ~]# kubectl get pods 
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-rj8sq   1/1     Running   0          7m33s
pc-deployment-5d89bdfbf9-ttwgg   1/1     Running   0          7m35s
pc-deployment-5d89bdfbf9-v4wvc   1/1     Running   0          7m34s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          3m31s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          3m31s

# 确保更新的pod没问题了，继续更新
[root@master ~]# kubectl rollout resume deploy pc-deployment 
deployment.apps/pc-deployment resumed

# 查看最后的更新情况
[root@master ~]# kubectl get rs  -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   0         0         0       21m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       16m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   4         4         4       5m11s   nginx        nginx:1.17.4   

[root@master ~]# kubectl get pods 
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6c9f56fcfb-7bfwh   1/1     Running   0          37s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-rf84v   1/1     Running   0          37s
```

### 5、删除Deployment

```powershell
# 删除deployment，其下的rs和pod也将被删除
[root@master ~]# kubectl delete -f pc-deployment.yaml
deployment.apps "pc-deployment" deleted
```

## 五、Statefulset

Statefulset主要是用来部署有状态应用

- 无状态应用中所有Pod之间是无法区分的，它们都是一样的，从而不需要考虑应用在哪个node上运行，能够进行随意伸缩和扩展。
- 有状态应用中每个Pod都是各不相同的，各自具有唯一的网络标识符和存储

```
对于StatefulSet中的Pod，每个Pod挂载自己独立的存储，如果一个Pod出现故障，从其他节点启动一个同样名字的Pod，要挂载上原来Pod的存储继续以它的状态提供服务。适合StatefulSet的业务包括数据库服务MySQL 和 PostgreSQL，集群化管理服务Zookeeper、etcd等有状态服务

使用StatefulSet，Pod仍然可以通过漂移到不同节点提供高可用，而存储也可以通过外挂的存储来提供高可靠性，StatefulSet做的只是将确定的Pod与确定的存储关联起来保证状态的连续性。
```

![image-20230315195248608](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5Clinux%E9%AB%98%E7%BA%A7---k8s%E4%B8%AD%E7%9A%84%E4%BA%94%E7%A7%8D%E6%8E%A7%E5%88%B6%E5%99%A8.assets%5Cimage-20230315195248608.png)

## 六、DaemonSet

```
DaemonSet 即后台支撑型服务，主要是用来部署守护进程。从而保证Pod运行在所有集群节点，或者是nodeSelector选定的全部节点。典型的后台支撑型服务包括：存储、日志和监控等。在每个节点上支撑K8S集群运行的服务。

守护进程在我们每个节点上，运行的是同一个pod，新加入的节点也同样运行在同一个pod里面

DaemonSet控制器的特点：
- 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
- 当节点从集群中移除时，Pod 也就被垃圾回收了
```

### 1、daemonset的资源清单文件

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

### 2、在配置清单中调用daemonset控制器

创建pc-daemonset.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: DaemonSet      
metadata:
  name: pc-daemonset
spec: 
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
```

```shell
# 创建daemonset
[root@master ~]# kubectl create -f  pc-daemonset.yaml
daemonset.apps/pc-daemonset created

# 查看daemonset
[root@master ~]#  kubectl get ds -o wide
NAME        DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE   AGE   CONTAINERS   IMAGES         
pc-daemonset   2        2        2      2           2        24s   nginx        nginx:1.17.1   

# 查看pod,发现在每个Node上都运行一个pod
[root@master ~]#  kubectl get pods -o wide
NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE    
pc-daemonset-9bck8   1/1     Running   0          37s   10.244.1.43   node1     
pc-daemonset-k224w   1/1     Running   0          37s   10.244.2.74   node2      

# 删除daemonset
[root@master ~]# kubectl delete -f pc-daemonset.yaml
daemonset.apps "pc-daemonset" deleted
```

## 七、Job

>Job Controller负责根据Job Spec创建Pod，并持续监控Pod的状态，直至其成功结束。如果失败，则根据restartPolicy（只支持OnFailure和Never，不支持Always）决定是否创建新的Pod再次重试任务。

![image-20230313200808482](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5Clinux%E9%AB%98%E7%BA%A7---k8s%E4%B8%AD%E7%9A%84%E4%BA%94%E7%A7%8D%E6%8E%A7%E5%88%B6%E5%99%A8.assets%5Cimage-20230313200808482.png)

```
Job负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束。

Kubernetes支持以下几种Job：
非并行Job：通常创建一个Pod直至其成功结束
固定结束次数的Job：设置.spec.completions，创建多个Pod，直到.spec.completions个Pod成功结束
带有工作队列的并行Job：设置.spec.Parallelism但不设置.spec.completions，当所有Pod结束并且至少一个成功时，Job就认为是成功
根据.spec.completions和.spec.Parallelism的设置，可以将Job划分为以下几种pattern
```

| JOB类型               | 使用实例                | 行为                                         | completions | parallelism |
| --------------------- | ----------------------- | -------------------------------------------- | ----------- | ----------- |
| 一次性Job             | 数据库迁移              | 创建一个Pod直至其成功结束                    | 1           | 1           |
| 固定结束次数的Job     | 处理工作队列的Pod       | 依次创建一个Pod运行直至completions个成功结束 | 2+          | 1           |
| 固定结束次数的并行Job | 多个Pod同时处理工作队列 | 依次创建多个Pod运行直至completions个成功结束 | 2+          | 2+          |
| 并行Job               | 多个Pod同时处理工作队列 | 创建一个或多个Pod直至有一个成功结束          | 1           | 2+          |

### 1、Job的资源清单文件

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定job需要成功运行Pods的次数。默认值: 1
  parallelism: 1 # 指定job在任一时刻应该并发运行Pods的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定job可运行的时间期限，超过时间还未结束，系统将会尝试进行终止。
  backoffLimit: 6 # 指定job失败后进行重试的次数。默认是6
  manualSelector: true # 是否可以使用selector选择器选择pod，默认是false
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或者OnFailure
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```

```markdown
关于重启策略设置的说明：    
如果指定为OnFailure，则job会在pod出现故障时重启容器，而不是创建pod，failed次数不变    
如果指定为Never，则job会在pod出现故障时创建新的pod，并且故障pod不会消失，也不会重启，failed次数加1    
如果指定为Always的话，就意味着一直重启，意味着job任务会重复去执行了，当然不对，所以不能设置为Always
```

### 2、在配置清单中调用Job控制器

创建pc-job.yaml，内容如下:

```yaml
apiVersion: batch/v1
kind: Job      
metadata:
  name: pc-job
spec:
  manualSelector: true
  selector:
    matchLabels:
      app: counter-pod
  template:
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

```shell
# 创建job
[root@master ~]# kubectl create -f pc-job.yaml
job.batch/pc-job created

# 查看job
[root@master ~]# kubectl get job  -o wide  -w
NAME     COMPLETIONS   DURATION   AGE   CONTAINERS   IMAGES         SELECTOR
pc-job   0/1           21s        21s   counter      busybox:1.30   app=counter-pod
pc-job   1/1           31s        79s   counter      busybox:1.30   app=counter-pod

# 通过观察pod状态可以看到，pod在运行完毕任务后，就会变成Completed状态
[root@master ~]# kubectl get pods  -w
NAME           READY   STATUS     RESTARTS      AGE
pc-job-rxg96   1/1     Running     0            29s
pc-job-rxg96   0/1     Completed   0            33s

# 接下来，调整下pod运行的总数量和并行数量 即：在spec下设置下面两个选项
#  completions: 6 # 指定job需要成功运行Pods的次数为6
#  parallelism: 3 # 指定job并发运行Pods的数量为3
#  然后重新运行job，观察效果，此时会发现，job会每次运行3个pod，总共执行了6个pod
[root@master ~]# kubectl get pods  -w
NAME           READY   STATUS    RESTARTS   AGE
pc-job-684ft   1/1     Running   0          5s
pc-job-jhj49   1/1     Running   0          5s
pc-job-pfcvh   1/1     Running   0          5s
pc-job-684ft   0/1     Completed   0          11s
pc-job-v7rhr   0/1     Pending     0          0s
pc-job-v7rhr   0/1     Pending     0          0s
pc-job-v7rhr   0/1     ContainerCreating   0          0s
pc-job-jhj49   0/1     Completed           0          11s
pc-job-fhwf7   0/1     Pending             0          0s
pc-job-fhwf7   0/1     Pending             0          0s
pc-job-pfcvh   0/1     Completed           0          11s
pc-job-5vg2j   0/1     Pending             0          0s
pc-job-fhwf7   0/1     ContainerCreating   0          0s
pc-job-5vg2j   0/1     Pending             0          0s
pc-job-5vg2j   0/1     ContainerCreating   0          0s
pc-job-fhwf7   1/1     Running             0          2s
pc-job-v7rhr   1/1     Running             0          2s
pc-job-5vg2j   1/1     Running             0          3s
pc-job-fhwf7   0/1     Completed           0          12s
pc-job-v7rhr   0/1     Completed           0          12s
pc-job-5vg2j   0/1     Completed           0          12s

# 删除job
[root@master ~]# kubectl delete -f pc-job.yaml
job.batch "pc-job" deleted
```



## 八、CronJob

```
CronJob 可以用来执行基于时间计划的定时任务，类似于Linux/Unix系统中的 crontable (opens new window)。
CronJob 执行周期性的重复任务时非常有用，例如备份数据、发送邮件等。CronJob 也可以用来指定将来某个时间点执行单个任务，例如将某项任务定时到系统负载比较低的时候执行。
一个 CronJob 对象就像 crontab (cron table) 文件中的一行。 它用Cron格式进行编写， 并周期性地在给定的调度时间执行 Job。

注意：
所有 CronJob 的 schedule: 时间都是基于kube-controller-manager. 的时区。
如果你的控制平面在 Pod 或是裸容器中运行了 kube-controller-manager， 那么为该容器所设置的时区将会决定 Cron Job 的控制器所使用的时区。
为 CronJob 资源创建清单时，请确保所提供的名称是一个合法的DNS 子域名. 名称不能超过 52 个字符。 这是因为 CronJob 控制器将自动在提供的 Job 名称后附加 11 个字符，并且存在一个限制， 即 Job 名称的最大长度不能超过 63 个字符。
CronJob 用于执行周期性的动作，例如备份、报告生成等。 这些任务中的每一个都应该配置为周期性重复的（例如：每天/每周/每月一次）； 你可以定义任务开始执行的时间间隔。
```

### 1、在配置清单中调用CronJob控制器

>下面的 CronJob 示例清单会在每分钟打印出当前时间和问候消息：

```shell
[root@master cronjob]# cat cronjob.yaml 
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello nihao
          restartPolicy: OnFailure
[root@master cronjob]# 
# 创建pod查看
[root@master cronjob]# kubectl apply -f cronjob.yaml 
cronjob.batch/hello created
[root@master cronjob]# 

# 等一分钟查看
[root@master cronjob]# kubectl get pod
NAME                          READY   STATUS      RESTARTS      AGE
hello-27978504-cddfx          0/1     Completed   0             40s

# 查看日志
[root@master cronjob]# kubectl logs hello-27978504-cddfx
Mon Mar 13 12:24:00 UTC 2023
Hello nihao
[root@master cronjob]# 
```

