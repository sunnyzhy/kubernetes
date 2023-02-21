# helm - redis 集群

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
# mkdir -p /usr/local/k8s/redis
```

## 查找 chart

```bash
# helm search repo redis
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/redis         	1.1.15       	4.0.8      	Open source, advanced key-value store. It is of...
aliyun/redis-ha      	2.0.1        	           	Highly available Redis cluster with multiple se...
bitnami/redis        	17.0.6       	7.0.4      	Redis(R) is an open source, advanced key-value ...
bitnami/redis-cluster	8.1.2        	7.0.4      	Redis(R) is an open source, scalable, distribut...
aliyun/sensu         	0.2.0        	           	Sensu monitoring framework backed by the Redis ...
```

## 拉取 chart

```bash
# helm pull bitnami/redis-cluster -d /usr/local/k8s/redis
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/redis/redis-cluster-8.1.2.tgz -C /usr/local/k8s/redis

# vim /usr/local/k8s/redis/redis-cluster/values.yaml
```

```yml
global:
  storageClass: "nfs-client"
  redis:
    password: "root"
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/redis/redis-cluster-8.1.2.tgz

# helm package /usr/local/k8s/redis/redis-cluster -d /usr/local/k8s/redis

# tree /usr/local/k8s/redis
/usr/local/k8s/redis
├── redis-cluster
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
│   ├── img
│   │   ├── redis-cluster-topology.png
│   │   └── redis-topology.png
│   ├── README.md
│   ├── templates
│   │   ├── configmap.yaml
│   │   ├── extra-list.yaml
│   │   ├── headless-svc.yaml
│   │   ├── _helpers.tpl
│   │   ├── metrics-prometheus.yaml
│   │   ├── metrics-svc.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── NOTES.txt
│   │   ├── poddisruptionbudget.yaml
│   │   ├── prometheusrule.yaml
│   │   ├── psp.yaml
│   │   ├── redis-rolebinding.yaml
│   │   ├── redis-role.yaml
│   │   ├── redis-serviceaccount.yaml
│   │   ├── redis-statefulset.yaml
│   │   ├── redis-svc.yaml
│   │   ├── scripts-configmap.yaml
│   │   ├── secret.yaml
│   │   ├── svc-cluster-external-access.yaml
│   │   ├── tls-secret.yaml
│   │   └── update-cluster.yaml
│   └── values.yaml
└── redis-cluster-8.1.2.tgz
```

## 部署

```bash
# helm install redis-cluster /usr/local/k8s/redis/redis-cluster-8.1.2.tgz -n iot
NAME: redis-cluster
LAST DEPLOYED: Tue Aug  2 14:18:13 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: redis-cluster
CHART VERSION: 8.1.2
APP VERSION: 7.0.4** Please be patient while the chart is being deployed **


To get your password run:
    export REDIS_PASSWORD=$(kubectl get secret --namespace "iot" redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)

You have deployed a Redis&reg; Cluster accessible only from within you Kubernetes Cluster.INFO: The Job to create the cluster will be created.To connect to your Redis&reg; cluster:

1. Run a Redis&reg; pod that you can use as a client:
kubectl run --namespace iot redis-cluster-client --rm --tty -i --restart='Never' \
 --env REDIS_PASSWORD=$REDIS_PASSWORD \
--image docker.io/bitnami/redis-cluster:7.0.4-debian-11-r1 -- bash

2. Connect using the Redis&reg; CLI:

redis-cli -c -h redis-cluster -a $REDIS_PASSWORD
```

查看资源:

```bash
# kubectl get statefulset -n iot | grep redis
redis-cluster   6/6     3m39s

# kubectl get pod -n iot -o wide | grep redis
redis-cluster-0                                    1/1     Running     1 (4m ago)     5m     10.244.1.114   centos-docker-164   <none>           <none>
redis-cluster-1                                    1/1     Running     1 (4m ago)     5m     10.244.2.96    centos-docker-165   <none>           <none>
redis-cluster-2                                    1/1     Running     1 (4m ago)     5m     10.244.1.115   centos-docker-164   <none>           <none>
redis-cluster-3                                    1/1     Running     1 (4m ago)     5m     10.244.2.98    centos-docker-165   <none>           <none>
redis-cluster-4                                    1/1     Running     1 (4m ago)     5m     10.244.2.97    centos-docker-165   <none>           <none>
redis-cluster-5                                    1/1     Running     1 (4m ago)     5m     10.244.1.116   centos-docker-164   <none>           <none>

# kubectl get pvc,pv -n iot | grep redis
persistentvolumeclaim/redis-data-redis-cluster-0   Bound    pvc-5b66c752-024b-4521-8df4-b326aec40c56   8Gi        RWO            nfs-client     5m2s
persistentvolumeclaim/redis-data-redis-cluster-1   Bound    pvc-1f89bcdd-f240-4dcb-9e30-c0d9bfa1160c   8Gi        RWO            nfs-client     5m2s
persistentvolumeclaim/redis-data-redis-cluster-2   Bound    pvc-90952c63-174f-41f8-b8e9-c4b61701341f   8Gi        RWO            nfs-client     5m2s
persistentvolumeclaim/redis-data-redis-cluster-3   Bound    pvc-f0fc22c2-5588-417a-924a-b574c77ec504   8Gi        RWO            nfs-client     5m2s
persistentvolumeclaim/redis-data-redis-cluster-4   Bound    pvc-89281be0-0b25-493a-8d7d-ac6c70a1f87c   8Gi        RWO            nfs-client     5m1s
persistentvolumeclaim/redis-data-redis-cluster-5   Bound    pvc-3d35cea4-0604-4fe3-9855-4d9088333054   8Gi        RWO            nfs-client     5m1s
persistentvolume/pvc-1f89bcdd-f240-4dcb-9e30-c0d9bfa1160c   8Gi        RWO            Delete           Bound    iot/redis-data-redis-cluster-1   nfs-client              5m2s
persistentvolume/pvc-3d35cea4-0604-4fe3-9855-4d9088333054   8Gi        RWO            Delete           Bound    iot/redis-data-redis-cluster-5   nfs-client              5m
persistentvolume/pvc-5b66c752-024b-4521-8df4-b326aec40c56   8Gi        RWO            Delete           Bound    iot/redis-data-redis-cluster-0   nfs-client              5m2s
persistentvolume/pvc-89281be0-0b25-493a-8d7d-ac6c70a1f87c   8Gi        RWO            Delete           Bound    iot/redis-data-redis-cluster-4   nfs-client              5m
persistentvolume/pvc-90952c63-174f-41f8-b8e9-c4b61701341f   8Gi        RWO            Delete           Bound    iot/redis-data-redis-cluster-2   nfs-client              5m2s
persistentvolume/pvc-f0fc22c2-5588-417a-924a-b574c77ec504   8Gi        RWO            Delete           Bound    iot/redis-data-redis-cluster-3   nfs-client              5m1s

# kubectl get svc -n iot | grep redis
redis-cluster            ClusterIP      10.107.235.251   <none>                                6379/TCP             5m58s
redis-cluster-headless   ClusterIP      None             <none>                                6379/TCP,16379/TCP   5m58s
```

### 集群内部解析 headless 服务

查询 pod 的域名:

```bash
# kubectl exec -it redis-cluster-0 -n iot -- /bin/sh
$ cat /etc/resolv.conf
search iot.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

通过 ```nslookup``` 命令解析 headless 服务对应的 pod IP:

```bash
# nslookup redis-cluster-headless.iot.svc.cluster.local $(kubectl get svc -n kube-system | grep 'kube-dns' | awk '{print $3}')
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	redis-cluster-headless.iot.svc.cluster.local
Address: 10.244.2.98
Name:	redis-cluster-headless.iot.svc.cluster.local
Address: 10.244.2.97
Name:	redis-cluster-headless.iot.svc.cluster.local
Address: 10.244.1.114
Name:	redis-cluster-headless.iot.svc.cluster.local
Address: 10.244.2.96
Name:	redis-cluster-headless.iot.svc.cluster.local
Address: 10.244.1.116
Name:	redis-cluster-headless.iot.svc.cluster.local
Address: 10.244.1.115
```

通过 ```dig``` 命令解析 headless 服务对应的 pod IP:

```bash
# dig redis-cluster-headless.iot.svc.cluster.local @$(kubectl get svc -n kube-system | grep 'kube-dns' | awk '{print $3}') +nocomments +noquestion +noauthority +noadditional +nostats

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7 <<>> redis-cluster-headless.iot.svc.cluster.local @10.96.0.10 +nocomments +noquestion +noauthority +noadditional +nostats
;; global options: +cmd
redis-cluster-headless.iot.svc.cluster.local. 14 IN A 10.244.2.97
redis-cluster-headless.iot.svc.cluster.local. 14 IN A 10.244.1.114
redis-cluster-headless.iot.svc.cluster.local. 14 IN A 10.244.2.96
redis-cluster-headless.iot.svc.cluster.local. 14 IN A 10.244.1.116
redis-cluster-headless.iot.svc.cluster.local. 14 IN A 10.244.2.98
redis-cluster-headless.iot.svc.cluster.local. 14 IN A 10.244.1.115
```

解析出来的 pod IP 对应各个 pod 在集群内部分配的 ip:

```bash
# kubectl get pods -l app.kubernetes.io/instance=redis-cluster -l app.kubernetes.io/name=redis-cluster -n iot -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}'
10.244.1.114
10.244.2.96
10.244.1.115
10.244.2.98
10.244.2.97
10.244.1.116
``

## 查看 redis 集群

```bash
# kubectl exec -it redis-cluster-0 -n iot -- /bin/sh

$ redis-cli -c

127.0.0.1:6379> cluster nodes
d03b29ecdf9072492d79f9aa102088544fac86f4 10.244.1.116:6379@16379 slave 033ebb2a5953959792c10e9c6c42cb14dc282376 0 1659424690701 2 connected
033ebb2a5953959792c10e9c6c42cb14dc282376 10.244.2.96:6379@16379 master - 0 1659424690000 2 connected 5461-10922
ee5fb9519167d2b67599a656b197f67080af4114 10.244.2.97:6379@16379 slave 9eda9611e40e07c43b4cf1d7c7d8c0940a62ebfa 0 1659424689695 1 connected
3f2678307ac8621e0e345591c1998f0e68d48d93 10.244.1.115:6379@16379 master - 0 1659424689000 3 connected 10923-16383
9eda9611e40e07c43b4cf1d7c7d8c0940a62ebfa 10.244.1.114:6379@16379 myself,master - 0 1659424689000 1 connected 0-5460
08d3df09c62784381b5edc9f637b74d04fad706a 10.244.2.98:6379@16379 slave 3f2678307ac8621e0e345591c1998f0e68d48d93 0 1659424691707 3 connected
```

## 外部访问 redis 集群

创建 NodePort 类型的 Service:

```bash
# vim /usr/local/k8s/redis/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-service
  namespace: iot
spec:
  selector:
    app.kubernetes.io/instance: redis-cluster
    app.kubernetes.io/name: redis-cluster
  ports:
    - protocol: TCP
      port: 6379
      targetPort: tcp-redis
      nodePort: 30379
  type: NodePort
```

```bash
# kubectl apply -f /usr/local/k8s/redis/service.yaml

# kubectl get svc redis-cluster-service -n iot
NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
redis-cluster-service   NodePort   10.108.226.8   <none>        6379:30379/TCP   12s
```

外部服务器连接 redis 集群:

***注: 示例仅查看 redis 集群节点，如果在在外部服务器执行 redis 命令，则需要给每个 redis-pod 创建单独的 service***

```bash
# redis-cli -h 192.168.5.163 -p 30379 -c

192.168.5.163:30379> cluster nodes
NOAUTH Authentication required.

192.168.5.163:30379> auth root
OK

192.168.5.163:30379> cluster nodes
08d3df09c62784381b5edc9f637b74d04fad706a 10.244.2.98:6379@16379 myself,slave 3f2678307ac8621e0e345591c1998f0e68d48d93 0 1659425255000 3 connected
9eda9611e40e07c43b4cf1d7c7d8c0940a62ebfa 10.244.1.114:6379@16379 master - 0 1659425257000 1 connected 0-5460
033ebb2a5953959792c10e9c6c42cb14dc282376 10.244.2.96:6379@16379 master - 0 1659425258000 2 connected 5461-10922
ee5fb9519167d2b67599a656b197f67080af4114 10.244.2.97:6379@16379 slave 9eda9611e40e07c43b4cf1d7c7d8c0940a62ebfa 0 1659425258445 1 connected
3f2678307ac8621e0e345591c1998f0e68d48d93 10.244.1.115:6379@16379 master - 0 1659425258000 3 connected 10923-16383
d03b29ecdf9072492d79f9aa102088544fac86f4 10.244.1.116:6379@16379 slave 033ebb2a5953959792c10e9c6c42cb14dc282376 0 1659425256000 2 connected
```

## FAQ

### (error) MOVED

- 原因: 一般是因为启动 ```redis-cli``` 时没有设置集群模式所导致
- 解决方法: 启动 ```redis-cli``` 时加上 ```-c``` 使用集群模式
   ```bash
   redis-cli -h <ip> -p <port> -c
   ```
