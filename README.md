# k8s-ingress-dashboard
通过Ingress访问kubernetes-dashboard，使用Cert-Manager通过ACME进行认证

简介

之前都是使用nodePort的方式暴露服务的，为了巩固ingress的知识，以及cert-manager的使用，所以想起来可以把Dashboard用ingress的方式暴露出去，这种方式我觉得可能会更优雅。

安装cert-manager

[Reference](https://cert-manager.io/docs/installation/kubectl/)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
```

安装成功，查看

```
root@master:~/deploy# kubectl get all -n cert-manager
NAME                                           READY   STATUS    RESTARTS      AGE
pod/cert-manager-64d9bc8b74-txszf              1/1     Running   1 (24d ago)   24d
pod/cert-manager-cainjector-6db6b64d5f-8qllw   1/1     Running   1 (24d ago)   24d
pod/cert-manager-webhook-6c9dd55dc8-z4qcs      1/1     Running   1 (24d ago)   24d

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.108.244.111   <none>        9402/TCP   24d
service/cert-manager-webhook   ClusterIP   10.103.16.48     <none>        443/TCP    24d

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           24d
deployment.apps/cert-manager-cainjector   1/1     1            1           24d
deployment.apps/cert-manager-webhook      1/1     1            1           24d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-64d9bc8b74              1         1         1       24d
replicaset.apps/cert-manager-cainjector-6db6b64d5f   1         1         1       24d
replicaset.apps/cert-manager-webhook-6c9dd55dc8      1         1         1       24d
```

安装ingress-nginx

[Reference](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)

使用wget下载下来，稍微做一些更改

```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/cloud/deploy.yaml
```

注释掉`externalTrafficPolicy`

```yaml
# externalTrafficPolicy: Local
```

替换镜像地址

`k8s.gcr.io`在国内无法访问，所以我把这两个镜像下载下来，上传到我的dockerhub账户下了，按照如下修改即可

deploy.yaml（[ingress-nginx.yaml](ingress-nginx.yaml)）

```bash
# image: k8s.gcr.io/ingress-nginx/controller:v1.2.0@sha256:d8196e3bc1e72547c5dec66d6556c0ff92a23f6d0919b206be170bc90d5f9185
image: xiebiao360/ingress-nginx-controller:v1.2.0

# image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
image: xiebiao360/kube-webhook-certgen:v1.1.1
```

注意，两个镜像共有3处要修改，勿遗漏

应用yaml配置

```bash
root@master:~/deploy# kubectl apply -f ./deploy.yaml 
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
```

查看部署情况

```bash
root@master:~/deploy# kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-mwhfk       0/1     Completed   0          6m2s
pod/ingress-nginx-admission-patch-wz7zf        0/1     Completed   3          6m2s
pod/ingress-nginx-controller-f686b4fd5-wgrrx   1/1     Running     0          6m2s

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.96.139.6     <pending>     80:30220/TCP,443:30613/TCP   6m2s
service/ingress-nginx-controller-admission   ClusterIP      10.102.103.33   <none>        443/TCP                      6m2s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           6m2s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-f686b4fd5   1         1         1       6m2s

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           74s        6m2s
job.batch/ingress-nginx-admission-patch    1/1           100s       6m2s
```

可以看到`service/ingress-nginx-controller`的外部IP`EXTERNAL-IP`值一直都是`<pending>`状态，这个值一般是由云服务提供商提供的（通常需要收费），不过我已经找到解决办法，后面会说

因为没有外部IP，所以在使用外网进行访问的时候，会提示`dial tcp xxx.xxx.xxx.xxx:80: connect: connection refused`

需要将本地流量转发到`ingress-nginx-controller`上进行处理

这里暂时通过`kubectl port-forward`将集群流量转发到`ingress-nginx-controller`

```bash
kubectl -n ingress-nginx --address 0.0.0.0 port-forward svc/ingress-nginx-controller 80
kubectl -n ingress-nginx --address 0.0.0.0 port-forward svc/ingress-nginx-controller 443
```

增加域名解析

（此处略过）

浏览器输入`http://domain:80`，不出意外的话会看到一个404页面，这是因为我们还没添加其他Ingress进行处理

创建ClusterIssuer和Certificate

[cluster-issuer.yaml](cluster-issuer.yaml)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: xiebiao360@outlook.com
    server: https://acme-v02.api.letsencrypt.org/directory
    preferredChain: "ISRG Root X1"
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dashboard-yameet-cn
  namespace: kubernetes-dashboard
spec:
  secretName: kubernetes-dashboard-certs
  secretTemplate:
    labels:
      k8s-app: kubernetes-dashboard
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  # subject:
  #   organizations:
  #     - jetstack
  dnsNames:
    - dashboard.yameet.cn
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

查看部署结果

```bash
root@master:~/deploy# kubectl apply -f ./cluster-issuer.yaml 
namespace/kubernetes-dashboard created
clusterissuer.cert-manager.io/letsencrypt-prod created
certificate.cert-manager.io/dashboard-foocloud-com-cn created
```

查看密钥信息

```
root@master:~/deploy# kubectl get secret -n kubernetes-dashboard
NAME                               TYPE                                  DATA   AGE
default-token-qtvwf                kubernetes.io/service-account-token   3      5h46m
kubernetes-dashboard-certs         kubernetes.io/tls                     2      5h45m
```

密钥已生成，接下来开始部署`kubernetes-dashboard`

[Reference](https://github.com/kubernetes/dashboard)

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml
```

老规矩，先下载下来，进行部分修改

[recommended.yaml](recommended.yaml)

```yaml
# 命名空间已经在之前创建过了，这里注释掉
# apiVersion: v1
# kind: Namespace
# metadata:
#   name: kubernetes-dashboard

---

# 之间的配置不用修改

---

# 这里的kubernetes-dashboard-certs已经交给Cert-Manager生成了，这里注释掉
# apiVersion: v1
# kind: Secret
# metadata:
#   labels:
#     k8s-app: kubernetes-dashboard
#   name: kubernetes-dashboard-certs
#   namespace: kubernetes-dashboard
# type: Opaque

---

# 其他配置不用修改
```

应用配置

```yaml
root@master:~/deploy# kubectl apply -f recommended.yaml 
namespace/kubernetes-dashboard unchanged
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

创建Ingress

[ingress-dashboard.yaml](ingress-dashboard.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-dashboard
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # 开启use-regex，启用path的正则匹配
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
    - hosts:
        - dashboard.yameet.cn
      secretName: kubernetes-dashboard-certs
  rules:
    - host: dashboard.yameet.cn
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: 
                name: kubernetes-dashboard
                port:
                  number: 443
```

应用配置

```bash
kubectl apply -f ingress-dashboard.yaml
```

查看结果

```
root@master:~/deploy# kubectl get all -n kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-799d786dbf-bk7pn   1/1     Running   0          5h34m
pod/kubernetes-dashboard-fb8648fd9-2mn8p         1/1     Running   0          5h34m

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/dashboard-metrics-scraper   ClusterIP   10.106.34.109   <none>        8000/TCP   5h34m
service/kubernetes-dashboard        ClusterIP   10.109.159.23   <none>        443/TCP    5h34m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           5h34m
deployment.apps/kubernetes-dashboard        1/1     1            1           5h34m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-799d786dbf   1         1         1       5h34m
replicaset.apps/kubernetes-dashboard-fb8648fd9         1         1         1       5h34m
```

创建示例用户

[dashboard-adminuser.yaml](dashboard-adminuser.yaml)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

```bash
root@master:~/deploy# kubectl apply -f dashboard-adminuser.yaml
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

获取token

```bash
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```



手动更改`EXTERNAL-IP`字段

`service/ingress-nginx-controller`的外部IP`EXTERNAL-IP`值一直都是`<pending>`状态，可以通过强行编辑该字段来解决

先查看`ingress-nginx-controller`部署在集群中的哪个节点上

```bash
root@master:~/deploy# kubectl get pod -n ingress-nginx -o wide
NAME                                       READY   STATUS      RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-r8tp6       0/1     Completed   0          9h    10.32.0.8   master   <none>           <none>
ingress-nginx-admission-patch-mdrkp        0/1     Completed   2          9h    10.32.0.7   master   <none>           <none>
ingress-nginx-controller-f686b4fd5-27rbm   1/1     Running     0          9h    10.32.0.8   master   <none>           <none>
```

获取该节点的内网主机IP

```bash
root@master:~/deploy# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:cd:e5:47 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.120/24 brd 10.0.0.255 scope global dynamic noprefixroute eth0
       valid_lft 29529478sec preferred_lft 29529478sec
    inet6 fe80::256d:54ed:9084:302a/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

上面的`10.0.0.120`即是

将`externalIPs`字段修改为该IP地址

```
kubectl edit -n ingress-nginx service/ingress-nginx-controller
```

```yaml
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.105.10.22
  clusterIPs:
    - 10.105.10.22
  externalIPs:
    - 10.0.0.120
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
```

修改`/etc/hosts`

```bash
echo "10.0.0.120 dashboard.yameet.cn" >> /etc/hosts
```

从浏览器访问`https://dashboard.yameet.cn`，应该可以了



参考文献：

https://www.youtube.com/watch?v=hoLUigg4V18

https://www.cnblogs.com/liyongjian5179/p/13204581.html

https://ceshiren.com/t/topic/16525

另外，可以关注下这个仓库[docker-development-youtube-series](https://github.com/marcel-dempers/docker-development-youtube-series)，里面有跟那个youtube视频配套的代码

