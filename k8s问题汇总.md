# k8s问题汇总

## 一、ImagePullBackOff问题

>可能存在的问题：
>
>私有镜像仓库认证失败
>
>镜像文件损坏
>
>**镜像拉取超时**
>
>镜像不存在

```shell
[root@master ~]# kubectl get pod -o wide
NAME                         READY   STATUS             RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
k8s-nginx-6d779d947c-jbkpc   0/1     ImagePullBackOff   0          14m   10.244.2.3   node3   <none>           <none>
k8s-nginx-6d779d947c-s4spt   1/1     Running            0          14m   10.244.1.2   node2   <none>           <none>
k8s-nginx-6d779d947c-ztkml   1/1     Running            0          14m   10.244.3.2   node1   <none>           <none>
```

>在这里经过检查发现是拉取超时的原因，配置镜像加速器就可以解决

## 二、删除一直处于terminating状态的namespace

```
https://www.cnblogs.com/fengdejiyixx/p/15186831.html
```

