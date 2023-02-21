# redis 集群

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

## 创建 nfs 的 rbac

参考 [rbac](./动态NFS.md 'rbac')

## 创建 nfs 存储

```bash
# mkdir -p /nfs/data/redis
```

## 创建 redis 配置目录

```bash
# mkdir -p /usr/local/k8s/redis
```

## 创建 StorageClass

```bash
# vim /usr/local/k8s/redis/nfs-storage.yaml
```

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: redis-nfs-storage
provisioner: redis/nfs
reclaimPolicy: Retain
```

```bash
# kubectl apply -f /usr/local/k8s/redis/nfs-storage.yaml
```

## 创建 nfs-client-provisioner

```bash
# vim /usr/local/k8s/redis/nfs-client-provisioner.yaml
```

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-nfs-client-provisioner
  labels:
    app: redis-nfs-client-provisioner
  namespace: iot
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: redis-nfs-client-provisioner
  template:
    metadata:
      labels:
        app: redis-nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: redis-nfs-client-provisioner
          image: gmoney23/nfs-client-provisioner:latest
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: redis/nfs
            - name: NFS_SERVER
              value: 192.168.5.163
            - name: NFS_PATH
              value: /nfs/data/redis
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.5.163
            path: /nfs/data/redis
```

```bash
# kubectl apply -f /usr/local/k8s/redis/nfs-client-provisioner.yaml
```

查看状态:

```bash
# kubectl get deployments -n iot | grep 'redis-nfs-client-provisioner'
redis-nfs-client-provisioner   1/1     1            1           25m

# kubectl get pods -n iot | grep 'redis-nfs-client-provisioner'
redis-nfs-client-provisioner-7f6854bd4b-hhmzm   1/1     Running     0          15m
```

## 创建 configmap

```bash
# wget -P /usr/local/k8s/redis https://download.redis.io/releases/redis-6.2.6.tar.gz

# tar -xzvf /usr/local/k8s/redis/redis-6.2.6.tar.gz -C /usr/local/k8s/redis

# cp /usr/local/k8s/redis/redis-6.2.6/redis.conf /usr/local/k8s/redis/redis.conf

# cp /usr/local/k8s/redis/redis-6.2.6/src/redis-trib.rb /usr/local/k8s/redis/redis-trib.rb

# rm -rf /usr/local/k8s/redis/redis-6.2.6*

# kubectl create configmap redis-conf --from-file=/usr/local/k8s/redis/redis.conf -n iot
```

## 创建 service

```bash
# vim /usr/local/k8s/redis/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
    appCluster: redis-cluster
  namespace: iot
spec:
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: cluster
    port: 16379
    targetPort: 16379
  selector:
    app: redis
    appCluster: redis-cluster
  clusterIP: None
```

```bash
# kubectl apply -f /usr/local/k8s/redis/service.yaml
```

查看状态:

```bash
# kubectl get services -n iot | grep 'redis-cluster'
redis-cluster   ClusterIP      None         <none>                                6379/TCP,16379/TCP   15m
```

## 创建 stateful-set

```bash
# vim /usr/local/k8s/redis/stateful-set.yaml
```

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: iot
  labels:
    app: redis
    appCluster: redis-cluster
spec:
  serviceName: redis-service
  replicas: 6
  selector:
    matchLabels:
      app: redis
      appCluster: redis-cluster
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:latest
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
          - "--appendonly"
          - "yes"
          - "--bind"
          - "0.0.0.0"
          - "--requirepass"
          - "123456"
          - "--cluster-enabled"
          - "yes"
          - "--cluster-announce-ip"
          - "$(POD_IP)"
        env:
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
          - name: TZ
            value: Asia/Shanghai
        ports:
            - name: redis
              containerPort: 6379
              protocol: TCP
            - name: cluster
              containerPort: 16379
              protocol: TCP
        volumeMounts:
          - name: redis-conf
            mountPath: /etc/redis
          - name: data
            mountPath: /var/lib/redis
      volumes:
      - name: redis-conf
        configMap:
          name: redis-conf
          items:
            - key: redis.conf
              path: redis.conf
  volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        volume.beta.kubernetes.io/storage-class: "redis-nfs-storage"
    spec:
      accessModes: 
        - ReadWriteMany
      resources:
        requests:
          storage: 200M
```

```bash
# kubectl apply -f /usr/local/k8s/redis/stateful-set.yaml
```

查看状态:

```bash
# kubectl get statefulsets -n iot | grep 'redis-cluster'
redis-cluster   6/6     15m

# kubectl get pods -n iot | grep 'redis-cluster'
redis-cluster-0                                 1/1     Running     0          15m
redis-cluster-1                                 1/1     Running     0          15m
redis-cluster-2                                 1/1     Running     0          15m
redis-cluster-3                                 1/1     Running     0          15m
redis-cluster-4                                 1/1     Running     0          15m
redis-cluster-5                                 1/1     Running     0          15m

# kubectl get pvc -n iot | grep 'redis-cluster'
data-redis-cluster-0   Bound    pvc-dcac552f-6a46-48de-a7b4-1580f503d9a5   200M       RWX            redis-nfs-storage     17m
data-redis-cluster-1   Bound    pvc-a2cdb7ce-7e67-4ac9-81fd-09fe5b9d5b2f   200M       RWX            redis-nfs-storage     16m
data-redis-cluster-2   Bound    pvc-ac9fc0c7-5fd4-4c4c-be58-c936e6bfe837   200M       RWX            redis-nfs-storage     16m
data-redis-cluster-3   Bound    pvc-295fc11e-8ee0-4bd6-977a-de4d8fd5c16f   200M       RWX            redis-nfs-storage     16m
data-redis-cluster-4   Bound    pvc-bbe6c35c-65a9-4c1e-9fe1-08ca31314acb   200M       RWX            redis-nfs-storage     16m
data-redis-cluster-5   Bound    pvc-5883e2fe-2201-45c2-9168-546c6b7b23af   200M       RWX            redis-nfs-storage     16m

# kubectl get pv -n iot | grep 'redis-cluster'
pvc-295fc11e-8ee0-4bd6-977a-de4d8fd5c16f   200M       RWX            Retain           Bound    iot/data-redis-cluster-3   redis-nfs-storage              17m
pvc-5883e2fe-2201-45c2-9168-546c6b7b23af   200M       RWX            Retain           Bound    iot/data-redis-cluster-5   redis-nfs-storage              17m
pvc-a2cdb7ce-7e67-4ac9-81fd-09fe5b9d5b2f   200M       RWX            Retain           Bound    iot/data-redis-cluster-1   redis-nfs-storage              17m
pvc-ac9fc0c7-5fd4-4c4c-be58-c936e6bfe837   200M       RWX            Retain           Bound    iot/data-redis-cluster-2   redis-nfs-storage              17m
pvc-bbe6c35c-65a9-4c1e-9fe1-08ca31314acb   200M       RWX            Retain           Bound    iot/data-redis-cluster-4   redis-nfs-storage              17m
pvc-dcac552f-6a46-48de-a7b4-1580f503d9a5   200M       RWX            Retain           Bound    iot/data-redis-cluster-0   redis-nfs-storage              17m
```

在任一物理机上执行以下命令查看 nfs 下的 redis 存储目录:

```bash
# tree /nfs/data/redis
/nfs/data/redis
├── iot-data-redis-cluster-0-pvc-dcac552f-6a46-48de-a7b4-1580f503d9a5
├── iot-data-redis-cluster-1-pvc-a2cdb7ce-7e67-4ac9-81fd-09fe5b9d5b2f
├── iot-data-redis-cluster-2-pvc-ac9fc0c7-5fd4-4c4c-be58-c936e6bfe837
├── iot-data-redis-cluster-3-pvc-295fc11e-8ee0-4bd6-977a-de4d8fd5c16f
├── iot-data-redis-cluster-4-pvc-bbe6c35c-65a9-4c1e-9fe1-08ca31314acb
└── iot-data-redis-cluster-5-pvc-5883e2fe-2201-45c2-9168-546c6b7b23af

6 directories, 0 files
```

## 初始化 Redis 集群

```bash
# kubectl get pods -n iot -o wide | grep 'redis-cluster'
redis-cluster-0                           1/1     Running   0               41m     10.244.2.59   centos-docker-165   <none>           <none>
redis-cluster-1                           1/1     Running   0               41m     10.244.1.72   centos-docker-164   <none>           <none>
redis-cluster-2                           1/1     Running   0               41m     10.244.2.60   centos-docker-165   <none>           <none>
redis-cluster-3                           1/1     Running   0               41m     10.244.1.73   centos-docker-164   <none>           <none>
redis-cluster-4                           1/1     Running   0               41m     10.244.1.74   centos-docker-164   <none>           <none>
redis-cluster-5                           1/1     Running   0               40m     10.244.2.61   centos-docker-165   <none>           <none>

# kubectl exec -it redis-cluster-0 -n iot -- redis-cli -a 123456 --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis -l appCluster=redis-cluster -n iot -o jsonpath='{range .items[*]}{.status.podIP}:6379 '{end})
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.244.1.74:6379 to 10.244.2.59:6379
Adding replica 10.244.2.61:6379 to 10.244.1.72:6379
Adding replica 10.244.1.73:6379 to 10.244.2.60:6379
M: c5b35ef4faa69f731701c2b8f758bae9f1f3d7c8 10.244.2.59:6379
   slots:[0-5460] (5461 slots) master
M: 720105715e152b325553aee31340889f88edd742 10.244.1.72:6379
   slots:[5461-10922] (5462 slots) master
M: 7fd1b6f805ea3b5084ad1bfa30159ac40c4f755d 10.244.2.60:6379
   slots:[10923-16383] (5461 slots) master
S: 7a63a847ebdaad42eb41544e5dbf56f563d3cffe 10.244.1.73:6379
   replicates 7fd1b6f805ea3b5084ad1bfa30159ac40c4f755d
S: 06fe4dd90a960fa4a1321b70603676041f99c038 10.244.1.74:6379
   replicates c5b35ef4faa69f731701c2b8f758bae9f1f3d7c8
S: ffa7a36d730963b621569116efef36d2752af2b5 10.244.2.61:6379
   replicates 720105715e152b325553aee31340889f88edd742
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 10.244.2.59:6379)
M: c5b35ef4faa69f731701c2b8f758bae9f1f3d7c8 10.244.2.59:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 7a63a847ebdaad42eb41544e5dbf56f563d3cffe 10.244.1.73:6379
   slots: (0 slots) slave
   replicates 7fd1b6f805ea3b5084ad1bfa30159ac40c4f755d
M: 7fd1b6f805ea3b5084ad1bfa30159ac40c4f755d 10.244.2.60:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 06fe4dd90a960fa4a1321b70603676041f99c038 10.244.1.74:6379
   slots: (0 slots) slave
   replicates c5b35ef4faa69f731701c2b8f758bae9f1f3d7c8
M: 720105715e152b325553aee31340889f88edd742 10.244.1.72:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: ffa7a36d730963b621569116efef36d2752af2b5 10.244.2.61:6379
   slots: (0 slots) slave
   replicates 720105715e152b325553aee31340889f88edd742
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

# kubectl exec -it redis-cluster-1 -n iot -- redis-cli -a 123456 -c cluster nodes
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
7a63a847ebdaad42eb41544e5dbf56f563d3cffe 10.244.1.73:6379@16379 slave 7fd1b6f805ea3b5084ad1bfa30159ac40c4f755d 0 1658820009634 3 connected
ffa7a36d730963b621569116efef36d2752af2b5 10.244.2.61:6379@16379 slave 720105715e152b325553aee31340889f88edd742 0 1658820011646 2 connected
7fd1b6f805ea3b5084ad1bfa30159ac40c4f755d 10.244.2.60:6379@16379 master - 0 1658820009000 3 connected 10923-16383
720105715e152b325553aee31340889f88edd742 10.244.1.72:6379@16379 myself,master - 0 1658820010000 2 connected 5461-10922
06fe4dd90a960fa4a1321b70603676041f99c038 10.244.1.74:6379@16379 slave c5b35ef4faa69f731701c2b8f758bae9f1f3d7c8 0 1658820010000 1 connected
c5b35ef4faa69f731701c2b8f758bae9f1f3d7c8 10.244.2.59:6379@16379 master - 0 1658820011000 1 connected 0-5460
```

## FAQ

### (error) MOVED

- 原因: 一般是因为启动 ```redis-cli``` 时没有设置集群模式所导致
- 解决方法: 启动 ```redis-cli``` 时加上 ```-c``` 使用集群模式
   ```bash
   redis-cli -h <ip> -p <port> -c
   ```
