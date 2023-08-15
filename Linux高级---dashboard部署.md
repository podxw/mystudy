# Linux高级---dashboard部署

[TOC]

## 一、下载安装dashboard

>先下载recommend.yaml文件
>
>```shell
>wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
>```

```shell
[root@master dashboard]# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
--2023-03-12 15:17:15--  https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
正在解析主机 raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.111.133, 185.199.110.133, ...
正在连接 raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：7621 (7.4K) [text/plain]
正在保存至: “recommended.yaml”

100%[===============================================================================>] 7,621       --.-K/s 用时 0s      

2023-03-12 15:17:17 (74.9 MB/s) - 已保存 “recommended.yaml” [7621/7621])

[root@master dashboard]# ls
```

## 二、修改相应的参数

>修改recommended.yaml中的service类型，默认是clustip，外面访问不了，将其修改成nodeport

```yaml
# 修改kubernetes-dashboard的Service类型
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort  # 新增
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30009  # 新增
  selector:
    k8s-app: kubernetes-dashboard
```

```shell
# 部署
[root@master ~]# kubectl create -f recommended.yaml


# 查看是否启动dashboard的pod，svc
[root@master dashboard]# kubectl get pod,svc -n kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-799d786dbf-rc2hj   1/1     Running   0          76s
pod/kubernetes-dashboard-546cbc58cd-dl757        1/1     Running   0          76s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.109.195.57   <none>        8000/TCP        76s
service/kubernetes-dashboard        NodePort    10.105.195.35   <none>        443:30009/TCP   76s


# 我们可以看到kubernetes-dashboard 和 kubernetes-dashboard都启动起来了，并且处于running状态
# kubernetes-dashboard 是dashboard自己的命名空间
```

## 三、创建访问用户，获取token

```powershell
# 创建账号
[root@master-1 ~]# kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard

# 授权
[root@master-1 ~]# kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin

# 获取账号token
[root@master ~]#  kubectl get secrets -n kubernetes-dashboard | grep dashboard-admin
dashboard-admin-token-mfs2w        kubernetes.io/service-account-token   3      2m35s

[root@master dashboard]# kubectl describe secrets dashboard-admin-token-mfs2w -n kubernetes-dashboard
Name:         dashboard-admin-token-mfs2w
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 8f657592-6328-4884-bb8a-a50a2d277ed5

Type:  kubernetes.io/service-account-token

Data
====
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlJCanJyMTluLWliRXF3aVA1LWRXcWtZbVlxdnhQa1Q5Y1EzT01nY3E1dGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tbWZzMnciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOGY2NTc1OTItNjMyOC00ODg0LWJiOGEtYTUwYTJkMjc3ZWQ1Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.s-g33KYjVWemyAccZ1zY6qNIrtBjuVPZ4bcDnyKiphkfvxD_dHhfljFY_OMGzNZcxanls9ujL_WlQ76au_lGzDrrIx2dRTw4gaHTnP3-c-lywn-deydOB5KC7Rgb7IRag8IShjWyqVSvImhW443Tf12CWEV3EhGx8pXUKrzx03Xytg-uuABbGSevQi1UXR31_iB15Jf_zKxeHcLZjffF8cEKyFPEKwSVFSNg_L3CwgcREGoB1GPplARrDeH0-Bm-x5kyoQljVHcR-KFdLFpnmnkI9p8TJR6Idrc3TP9-TF3xjX6z_W8Xccamz0gFgzi_6h2jvOTfVNTFUBN3ONRtnw
ca.crt:     1099 bytes
[root@master dashboard]# 
```

## 四、测试访问

https://192.168.17.144:30009

> 输入上面获得的token，然后登录即可

![image-20230601113731177](C:%5C%E4%B8%89%E5%88%9B%E5%9F%B9%E8%AE%AD%5CLinux%E9%AB%98%E7%BA%A7%5Clinux%E9%AB%98%E7%BA%A7%E7%AC%94%E8%AE%B0--k8s%5CLinux%E9%AB%98%E7%BA%A7---dashboard%E9%83%A8%E7%BD%B2.assets%5Cimage-20230601113731177.png)

