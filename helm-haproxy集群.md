# helm - haproxy 集群

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

## 创建 redis 目录

```bash
# mkdir -p /usr/local/k8s/haproxy
```

## 查找 chart

```bash
# helm search repo haproxy
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/haproxy      	0.6.2        	2.6.6      	HAProxy is a TCP proxy and a HTTP reverse proxy...
bitnami/haproxy-intel	0.2.7        	2.6.6      	HAProxy is a high-performance, open-source load...
```

## 拉取 chart

```bash
# helm pull bitnami/haproxy -d /usr/local/k8s/haproxy
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/haproxy/haproxy-0.6.2.tgz -C /usr/local/k8s/haproxy

# sed -i 's+kind: Deployment+kind: StatefulSet+' /usr/local/k8s/haproxy/haproxy/templates/deployment.yaml

# sed -i 's+{{- if .Values.clusterIP }}+{{- if .Values.service.clusterIP }}+' /usr/local/k8s/haproxy/haproxy/templates/service.yaml

# sed -i 's+clusterIP: {{ .Values.clusterIP }}+clusterIP: {{ .Values.service.clusterIP }}+' /usr/local/k8s/haproxy/haproxy/templates/service.yaml

# vim /usr/local/k8s/haproxy/haproxy/values.yaml
```

```yml
global:
  storageClass: "nfs-client"

service:
  type: ClusterIP
  clusterIP: None

ingress:
  enabled: true
  hostname: iot.haproxy
  annotations: 
    kubernetes.io/ingress.class: nginx

replicaCount: 3

tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"

extraEnvVars:
  - name: TZ
    value: Asia/Shanghai
```

注：把 deployment 

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/haproxy/haproxy-0.6.2.tgz

# helm package /usr/local/k8s/haproxy/haproxy -d /usr/local/k8s/haproxy

# tree /usr/local/k8s/haproxy
/usr/local/k8s/haproxy
├── haproxy
│   ├── Chart.lock
│   ├── charts
│   │   └── common
│   │       ├── Chart.yaml
│   │       ├── README.md
│   │       ├── templates
│   │       │   ├── _affinities.tpl
│   │       │   ├── _capabilities.tpl
│   │       │   ├── _errors.tpl
│   │       │   ├── _images.tpl
│   │       │   ├── _ingress.tpl
│   │       │   ├── _labels.tpl
│   │       │   ├── _names.tpl
│   │       │   ├── _secrets.tpl
│   │       │   ├── _storage.tpl
│   │       │   ├── _tplvalues.tpl
│   │       │   ├── _utils.tpl
│   │       │   ├── validations
│   │       │   │   ├── _cassandra.tpl
│   │       │   │   ├── _mariadb.tpl
│   │       │   │   ├── _mongodb.tpl
│   │       │   │   ├── _mysql.tpl
│   │       │   │   ├── _postgresql.tpl
│   │       │   │   ├── _redis.tpl
│   │       │   │   └── _validations.tpl
│   │       │   └── _warnings.tpl
│   │       └── values.yaml
│   ├── Chart.yaml
│   ├── README.md
│   ├── templates
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── extra-list.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   ├── pdb.yaml
│   │   ├── service-account.yaml
│   │   └── service.yaml
│   └── values.yaml
└── haproxy-0.6.2.tgz
```

## 部署

```bash
# helm install haproxy-cluster /usr/local/k8s/haproxy/haproxy-0.6.2.tgz -n iot
NAME: haproxy-cluster
LAST DEPLOYED: Wed Nov 23 13:59:06 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: haproxy
CHART VERSION: 0.6.2
APP VERSION: 2.6.6

** Please be patient while the chart is being deployed **
1. HAproxy has been started. You can find out the port numbers being used by HAProxy by running:

    $ kubectl describe svc haproxy-cluster --namespace iot
```

查看资源:

```bash
# kubectl get statefulset -n iot | grep haproxy
haproxy-cluster                   3/3     3            3           114s

# kubectl get pod -n iot -o wide | grep haproxy
haproxy-cluster-0                   1/1     Running            0                 2m13s   10.244.0.92    centos-docker-163   <none>           <none>
haproxy-cluster-1                   1/1     Running            0                 2m13s   10.244.2.85    centos-docker-165   <none>           <none>
haproxy-cluster-2                   1/1     Running            0                 2m13s   10.244.1.30    centos-docker-164   <none>           <none>

# kubectl get svc -n iot | grep haproxy
haproxy-cluster                         ClusterIP   None             <none>        80/TCP                                                                        2m46s
```

### 集群内部解析 headless 服务

查询 pod 的域名:

```bash
# kubectl exec -it haproxy-cluster-0 -n iot -- /bin/sh
$ cat /etc/resolv.conf
search iot.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

通过 ```nslookup``` 命令解析 headless 服务对应的 pod IP:

```bash
# nslookup haproxy-cluster.iot.svc.cluster.local $(kubectl get svc -n kube-system | grep 'kube-dns' | awk '{print $3}')
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	haproxy-cluster.iot.svc.cluster.local
Address: 10.244.2.87
Name:	haproxy-cluster.iot.svc.cluster.local
Address: 10.244.1.32
Name:	haproxy-cluster.iot.svc.cluster.local
Address: 10.244.0.94
```

通过 ```dig``` 命令解析 headless 服务对应的 pod IP:

```bash
# dig haproxy-cluster.iot.svc.cluster.local @$(kubectl get svc -n kube-system | grep 'kube-dns' | awk '{print $3}') +nocomments +noquestion +noauthority +noadditional +nostats

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7 <<>> haproxy-cluster.iot.svc.cluster.local @10.96.0.10 +nocomments +noquestion +noauthority +noadditional +nostats
;; global options: +cmd
haproxy-cluster.iot.svc.cluster.local. 3 IN A	10.244.2.87
haproxy-cluster.iot.svc.cluster.local. 3 IN A	10.244.0.94
haproxy-cluster.iot.svc.cluster.local. 3 IN A	10.244.1.32
```

解析出来的 pod IP 对应各个 pod 在集群内部分配的 ip:

```bash
# kubectl get pods -l app.kubernetes.io/instance=haproxy-cluster -l app.kubernetes.io/name=haproxy -n iot -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}'
10.244.0.94
10.244.1.32
10.244.2.87
``

## 外部访问 haproxy 集群

在任一外部服务器的 hosts 文件里配置域名映射:

```bash
# vim /etc/hosts
192.168.5.165 iot.haproxy
```

在浏览器的地址栏里输入:```http://iot.haproxy/monitor```
