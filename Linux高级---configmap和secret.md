# Linux高级---configmap和secret

[TOC]

## 一、ConfigMap

![image-20230522203903073](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---configmap%E5%92%8Csecret.assets%5Cimage-20230522203903073.png)

### 1、介绍

>ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中。使用时， [Pods](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。
>
>ConfigMap 将你的环境配置信息和 [容器镜像](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-image) 解耦，便于应用配置的修改。

**注意：**

ConfigMap 并不提供保密或者加密功能。 如果你想存储的数据是机密的，请使用 [Secret](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/)， 或者使用其他第三方工具来保证你的数据的私密性，而不是用 ConfigMap。

**使用**

你可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

1. 在容器命令和参数内
2. 容器的环境变量
3. 在只读卷里面添加一个文件，让应用来读取
4. 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

这些不同的方法适用于不同的数据使用方式。 对前三个方法，[kubelet](https://kubernetes.io/docs/reference/generated/kubelet) 使用 ConfigMap 中的数据在 Pod 中启动容器。第四种方法意味着你必须编写代码才能读取 ConfigMap 和它的数据。

### 2、创建configmap

```powershell
[root@master demo]# vi configMap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
        name: test-config
    data:
        username: zhangsan
        password: yuanke
        username: lisi

[root@master demo]# kubectl create -f configMap.yaml
configmap/test-config created
[root@master demo]# vi configMap.yaml
[root@master demo]# kubectl get configMaps
NAME          DATA   AGE
test-config   2      45s
[root@master demo]# kubectl describe configmaps test-config
Name:         test-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
password:
----
yuanke
username:
----
lisi
Events:  <none>
```

### 3、使用configmap

> vim test-configMap-env-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap-env-pod
spec:
  containers:
     - name: test-container
       image: radial/busyboxplus
       imagePullPolicy: IfNotPresent
       command: ["/bin/sh","-c","sleep 1000000"]
       envFrom:
       - configMapRef:
            name: test-config
```

```powershell
[root@master demo]# kubectl create -f test-configMap-env-pod.yaml
pod/test-configmap-env-pod created
[root@master demo]# kubectl get pod
NAME                                  READY   STATUS      RESTARTS   AGE
test-configmap-env-pod                1/1     Running     0          42s
[root@master demo]# kubectl exec -it test-configmap-env-pod -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=test-configmap-env-pod
TERM=xterm
username=lisi
password=yuanke
```

### 4、引入环境变量的另一种方式

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap-env-pod
spec:
  containers:
     - name: test-container
       image: radial/busyboxplus
       imagePullPolicy: IfNotPresent
       command: ["/bin/sh","-c","echo ${MYSQLUSER} ${MYSQLPASSWD};sleep 1000000"]
       env:
       - name: MYSQLUSER
         valueFrom:
            configMapKeyRef:
               name: test-config
               key: username
       - name: MYSQLPASSWD
         valueFrom:
            configMapKeyRef:
               name: test-config
               key: password
```

```powershell
[root@master demo]# kubectl create -f test-configMap-env-pod.yaml
pod/test-configmap-env-pod created
[root@master demo]# kubectl get pod
NAME                                  READY   STATUS      RESTARTS   AGE
test-configmap-env-pod                1/1     Running     0          5s
[root@master demo]# kubectl exec -it test-configmap-env-pod -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=test-configmap-env-pod
TERM=xterm
MYSQLUSER=lisi
MYSQLPASSWD=yuanke
```

## 二、Secret

### 1、介绍

>Secret 是一种包含少量敏感信息例如密码、令牌或密钥的对象。 这样的信息可能会被放在 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 规约中或者镜像中。 使用 Secret 意味着你不需要在应用程序代码中包含机密数据。
>
>由于创建 Secret 可以独立于使用它们的 Pod， 因此在创建、查看和编辑 Pod 的工作流程中暴露 Secret（及其数据）的风险较小。 Kubernetes 和在集群中运行的应用程序也可以对 Secret 采取额外的预防措施， 例如避免将机密数据写入非易失性存储。
>
>Secret 类似于 [ConfigMap](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-pod-configmap/) 但专门用于保存机密数据。

**注意：**

默认情况下，Kubernetes Secret 未加密地存储在 API 服务器的底层数据存储（etcd）中。 任何拥有 API 访问权限的人都可以检索或修改 Secret，任何有权访问 etcd 的人也可以。 此外，任何有权限在命名空间中创建 Pod 的人都可以使用该访问权限读取该命名空间中的任何 Secret； 这包括间接访问，例如创建 Deployment 的能力。

为了安全地使用 Secret，请至少执行以下步骤：

1. 为 Secret [启用静态加密](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/encrypt-data/)。
2. 以最小特权访问 Secret 并[启用或配置 RBAC 规则](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/)。
3. 限制 Secret 对特定容器的访问。
4. [考虑使用外部 Secret 存储驱动](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver)。

**Secret 的使用**

Pod 可以用三种方式之一来使用 Secret：

- 作为挂载到一个或多个容器上的[卷](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/) 中的[文件](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。
- 作为[容器的环境变量](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)。
- 由 [kubelet 在为 Pod 拉取镜像时使用](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#using-imagepullsecrets)。

### 2、创建secret

您也可以先以 json 或 yaml 格式在文件中创建一个 secret 对象，然后创建该对象。

每一项必须是 base64 编码：

```shell
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

解密：

```shell
echo 'YWRtaW4=' | base64 --decode
返回admin
```

vim secret-env.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret-env
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

```powershell
[root@master demo]# kubectl create -f secret-env.yaml
secret/mysecret-env created
[root@master demo]# kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-mp2h9   kubernetes.io/service-account-token   3      21d
mysecret-env          Opaque                                2      10s
tls-secret            kubernetes.io/tls                     2      23h
```

### 3、使用secret

vim secret-pod-env1.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-secret
spec:
  containers:
     - name: test-nginx
       image: nginx
       envFrom:
       - secretRef:
            name: mysecret-env
```

```shell
[root@master secret]# kubectl apply -f secret-pod-env1.yaml 
pod/envfrom-secret created
[root@master secret]# kubectl get pod 
NAME                                READY   STATUS    RESTARTS   AGE
envfrom-secret                      1/1     Running   0          16s
[root@master secret]# kubectl exec -it envfrom-secret -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=envfrom-secret
TERM=xterm
password=1f2d1e2e67df
username=admin
```

### 4、引入环境变量的另一种方式

vim  secret-pod-env2.yaml 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secret-env-pod
spec:
  containers:
     - name: test-container
       image: radial/busyboxplus
       imagePullPolicy: IfNotPresent
       command: ["/bin/sh","-c","echo ${MYSQLUSER} ${MYSQLPASSWD};sleep 1000000"]
       env:
       - name: MYSQLUSER
         valueFrom:
            secretKeyRef:
               name: mysecret-env
               key: username
       - name: MYSQLPASSWD
         valueFrom:
            secretKeyRef:
               name: mysecret-env
               key: password
```

```shell
[root@master secret]# kubectl apply -f secret-pod-env2.yaml 
pod/test-secret-env-pod created
[root@master secret]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
test-secret-env-pod                 1/1     Running   0          5s
[root@master secret]# kubectl exec -it test-secret-env-pod -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=test-secret-env-pod
TERM=xterm
MYSQLUSER=admin
MYSQLPASSWD=1f2d1e2e67df
```

