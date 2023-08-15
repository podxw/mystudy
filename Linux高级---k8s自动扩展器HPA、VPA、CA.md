# Linux高级---k8s自动扩展器HPA、VPA、CA

[TOC]

## 一、Kubernetes Pod水平自动伸缩（HPA）

>HPA官方文档 ：https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/

### 1、HPA简介

- HAP，全称 Horizontal Pod Autoscaler， 可以基于 CPU 利用率自动扩缩 ReplicationController、Deployment 和 ReplicaSet 中的 Pod 数量。 除了 CPU 利用率，也可以基于其他应程序提供的自定义度量指标来执行自动扩缩。 Pod 自动扩缩不适用于无法扩缩的对象，比如 DaemonSet。
- Pod 水平自动扩缩特性由 Kubernetes API 资源和控制器实现。资源决定了控制器的行为。 控制器会周期性的调整副本控制器或 Deployment 中的副本数量，以使得 Pod 的平均 CPU 利用率与用户所设定的目标值匹配。

![image-20230312155256082](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230312155256082.png)

- HPA 定期检查内存和 CPU 等指标，自动调整 Deployment 中的副本数，比如流量变化：

![image-20230312155320464](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230312155320464.png)

>实际生产中，广泛使用这四类指标：
>1、Resource metrics - CPU核内存利用率指标
>2、Pod metrics - 例如网络利用率和流量
>3、Object metrics - 特定对象的指标，比如Ingress, 可以按每秒使用请求数来扩展容器
>4、Custom metrics - 自定义监控，比如通过定义服务响应时间，当响应时间达到一定指标时自动扩容

## 二、Kubernetes Pod垂直自动伸缩（VPA）

>VPA项目托管地址 ：https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

### 1、VPA 简介

- VPA 全称 Vertical Pod Autoscaler，即垂直 Pod 自动扩缩容，它根据容器资源使用率自动设置 CPU 和 内存 的requests，从而允许在节点上进行适当的调度，以便为每个 Pod 提供适当的资源。
- 它既可以缩小过度请求资源的容器，也可以根据其使用情况随时提升资源不足的容量。

![image-20230312155537910](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230312155537910.png)

- 有些时候无法通过增加 Pod 数来扩容，比如数据库。这时候可以通过 VPA 增加 Pod 的大小，比如调整 Pod 的 CPU 和内存：

![image-20230312155558860](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230312155558860.png)

```
限制
不能与HPA（Horizontal Pod Autoscaler ）一起使用；
优势
Pod 资源用其所需，所以集群节点使用效率高；
Pod 会被安排到具有适当可用资源的节点上；
不必运行基准测试任务来确定 CPU 和内存请求的合适值；
VPA 可以随时调整 CPU 和内存请求，无需人为操作，因此可以减少维护时间；
```

## 三、Kubernetes 集群自动缩放器（CA）

>CA项目托管地址 ：https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

### 1、CA简介

- 集群自动伸缩器（CA）基于待处理的豆荚扩展集群节点。它会定期检查是否有任何待处理的豆荚，如果需要更多的资源，并且扩展的集群仍然在用户提供的约束范围内，则会增加集群的大小。CA与云供应商接口，请求更多节点或释放空闲节点。它与GCP、AWS和Azure兼容。版本1.0（GA）与Kubernetes 1.8一起发布。

![image-20230312155915813](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230312155915813.png)

- 当集群资源不足时，CA 会自动配置新的计算资源并添加到集群中：

![image-20230312155934591](C:%5CUsers%5C%E8%82%96%E5%A8%81%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20230312155934591.png)