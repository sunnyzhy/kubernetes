# helm - harbor 集群

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

- harbor 账号的密码长度必须 ```>= 10```

## 创建 harbor 目录

```bash
# mkdir -p /usr/local/k8s/harbor
```

## 查找 chart

```bash
# helm search repo harbor
NAME          	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/harbor	15.0.5       	2.5.3      	Harbor is an open source trusted cloud-native r...
```

## 拉取 chart

```bash
# helm pull bitnami/harbor -d /usr/local/k8s/harbor
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/harbor/harbor-15.0.5.tgz -C /usr/local/k8s/harbor

# vim /usr/local/k8s/harbor/harbor/values.yaml
```

```yml
global:
  storageClass: "nfs-client"

adminPassword: "adminpassword"

exposureType: ingress

ingress:
  core:
    ingressClassName: "nginx"
    hostname: core.harbor.domain

service:
  type: ClusterIP
```

***注: 如果使用 harbor 自带证书的话， 就不要修改 ingress 默认的 hostname，因为 harbor 自带证书的 DNS 是 ```core.harbor.domain```***

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/harbor/harbor-15.0.5.tgz

# helm package /usr/local/k8s/harbor/harbor -d /usr/local/k8s/harbor
```

## 部署

```bash
# helm install harbor /usr/local/k8s/harbor/harbor-15.0.5.tgz
NAME: harbor
LAST DEPLOYED: Fri Aug 26 14:38:42 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: harbor
CHART VERSION: 15.0.5
APP VERSION: 2.5.3

** Please be patient while the chart is being deployed **

1. Get the Harbor URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w harbor'
    export SERVICE_IP=$(kubectl get svc --namespace default harbor --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
    echo "Harbor URL: http://$SERVICE_IP/"

2. Login with the following credentials to see your Harbor application

  echo Username: "admin"
  echo Password: $(kubectl get secret --namespace default harbor-core-envvars -o jsonpath="{.data.HARBOR_ADMIN_PASSWORD}" | base64 -d)
```

查看资源:

```bash
# kubectl get deploy | grep harbor
harbor-chartmuseum     1/1     1            1           2m39s
harbor-core            1/1     1            1           2m39s
harbor-jobservice      1/1     1            1           2m39s
harbor-notary-server   1/1     1            1           2m39s
harbor-notary-signer   1/1     1            1           2m39s
harbor-portal          1/1     1            1           2m39s
harbor-registry        1/1     1            1           2m39s

# kubectl get sts | grep harbor
harbor-postgresql     1/1     3m4s
harbor-redis-master   1/1     3m4s
harbor-trivy          1/1     3m4s

# kubectl get pod -o wide | grep harbor
harbor-chartmuseum-6565cf975-spbsp      1/1     Running   0          3m15s   10.244.1.68   centos-docker-164   <none>           <none>
harbor-core-7f9fd89f4b-hv8lk            1/1     Running   0          3m15s   10.244.2.72   centos-docker-165   <none>           <none>
harbor-jobservice-75b4544c8d-qwzvf      1/1     Running   0          3m15s   10.244.1.69   centos-docker-164   <none>           <none>
harbor-notary-server-f55667676-vqrrc    1/1     Running   0          3m15s   10.244.2.73   centos-docker-165   <none>           <none>
harbor-notary-signer-5d5fcdf6fb-sbzqs   1/1     Running   0          3m15s   10.244.2.74   centos-docker-165   <none>           <none>
harbor-portal-587bbbcbfc-khwcw          1/1     Running   0          3m15s   10.244.1.70   centos-docker-164   <none>           <none>
harbor-postgresql-0                     1/1     Running   0          3m15s   10.244.1.72   centos-docker-164   <none>           <none>
harbor-redis-master-0                   1/1     Running   0          3m15s   10.244.1.71   centos-docker-164   <none>           <none>
harbor-registry-cb4bfbc96-hqst7         2/2     Running   0          3m15s   10.244.2.76   centos-docker-165   <none>           <none>
harbor-trivy-0                          1/1     Running   0          3m15s   10.244.2.75   centos-docker-165   <none>           <none>

# kubectl get pvc,pv | grep harbor
persistentvolumeclaim/data-harbor-postgresql-0           Bound    pvc-3bc1f6e2-6d1d-458d-8ac7-09ae68403e9d   8Gi        RWO            nfs-client     3m36s
persistentvolumeclaim/data-harbor-trivy-0                Bound    pvc-a9bb7ff8-0ad1-4f2f-8141-61d0aae959f5   5Gi        RWO            nfs-client     3m36s
persistentvolumeclaim/harbor-chartmuseum                 Bound    pvc-bff6415e-fc39-4a07-9995-26696efe8aec   5Gi        RWO            nfs-client     3m36s
persistentvolumeclaim/harbor-jobservice                  Bound    pvc-5aa16c97-d84c-4566-b893-37eaee9ee6ba   1Gi        RWO            nfs-client     3m36s
persistentvolumeclaim/harbor-registry                    Bound    pvc-9fd5a5c4-057a-488e-b96b-98631b524ab0   5Gi        RWO            nfs-client     3m36s
persistentvolumeclaim/redis-data-harbor-redis-master-0   Bound    pvc-465b4c99-3a26-4ab0-9f5a-5d576b7fcf62   8Gi        RWO            nfs-client     3m36s
persistentvolume/pvc-3bc1f6e2-6d1d-458d-8ac7-09ae68403e9d   8Gi        RWO            Delete           Bound    default/data-harbor-postgresql-0                  nfs-client              3m36s
persistentvolume/pvc-465b4c99-3a26-4ab0-9f5a-5d576b7fcf62   8Gi        RWO            Delete           Bound    default/redis-data-harbor-redis-master-0          nfs-client              3m36s
persistentvolume/pvc-5aa16c97-d84c-4566-b893-37eaee9ee6ba   1Gi        RWO            Delete           Bound    default/harbor-jobservice                         nfs-client              3m36s
persistentvolume/pvc-9fd5a5c4-057a-488e-b96b-98631b524ab0   5Gi        RWO            Delete           Bound    default/harbor-registry                           nfs-client              3m36s
persistentvolume/pvc-a9bb7ff8-0ad1-4f2f-8141-61d0aae959f5   5Gi        RWO            Delete           Bound    default/data-harbor-trivy-0                       nfs-client              3m36s
persistentvolume/pvc-bff6415e-fc39-4a07-9995-26696efe8aec   5Gi        RWO            Delete           Bound    default/harbor-chartmuseum                        nfs-client              3m36s

# kubectl get svc | grep harbor
harbor-chartmuseum      ClusterIP   10.109.157.80    <none>        80/TCP              3m55s
harbor-core             ClusterIP   10.101.227.26    <none>        80/TCP              3m54s
harbor-jobservice       ClusterIP   10.97.5.106      <none>        80/TCP              3m54s
harbor-notary-server    ClusterIP   10.99.101.50     <none>        4443/TCP            3m54s
harbor-notary-signer    ClusterIP   10.101.234.132   <none>        7899/TCP            3m54s
harbor-portal           ClusterIP   10.108.11.110    <none>        80/TCP              3m54s
harbor-postgresql       ClusterIP   10.109.203.205   <none>        5432/TCP            3m55s
harbor-postgresql-hl    ClusterIP   None             <none>        5432/TCP            3m55s
harbor-redis-headless   ClusterIP   None             <none>        6379/TCP            3m55s
harbor-redis-master     ClusterIP   10.105.103.143   <none>        6379/TCP            3m55s
harbor-registry         ClusterIP   10.98.78.73      <none>        5000/TCP,8080/TCP   3m54s
harbor-trivy            ClusterIP   10.98.194.224    <none>        8080/TCP            3m54s

# kubectl get ingress | grep harbor
harbor-ingress          nginx    core.harbor.domain     10.108.27.226   80      4m15s
harbor-ingress-notary   <none>   notary.harbor.domain                   80      4m15s
```

## 内部访问 harbor 集群

略

## 外部访问 harbor 集群

### 通过对外 Service 的方式

略

### 通过 ingress 的方式

在任一外部服务器的 hosts 文件里配置域名映射:

```bash
# vim /etc/hosts
192.168.5.165 core.harbor.domain
```

在浏览器的地址栏里输入:```https://core.harbor.domain/```, ```用户名/密码: admin/adminpassword```
