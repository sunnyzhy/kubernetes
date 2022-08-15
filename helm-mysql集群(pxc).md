# helm - mysql 集群 (pxc)

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

- 集群相关资料:

    1. Oracle MySQL
        ```
        https://github.com/mysql/mysql-operator
        https://dev.mysql.com/doc/mysql-operator/en/
        ```
 
    2. Percona MySQL
        ```
        https://github.com/percona/percona-xtradb-cluster-operator
        https://www.percona.com/doc/kubernetes-operator-for-pxc/helm.html
        ```

    3. presslabs/mysql-operator
        ```
        https://github.com/bitpoke/mysql-operator
        ```

    4. Operator社区网站：
        ```
        https://operatorhub.io/?category=Database
        ```

## 创建 mysql 目录

```bash
# mkdir -p /usr/local/k8s/mysql
```

## 添加 chart 源

```bash
# helm repo add percona https://percona.github.io/percona-helm-charts/

# helm repo update
```

## 查找 chart

```bash
# helm search repo percona
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/percona               	0.3.0        	           	free, fully compatible, enhanced, open source d...
aliyun/percona-xtradb-cluster	0.0.2        	5.7.19     	free, fully compatible, enhanced, open source d...
percona/pg-db                	1.2.0        	1.2.0      	A Helm chart for Deploying the Percona PostgreS...
percona/pg-operator          	1.2.0        	1.2.0      	A Helm chart to deploy the Percona Distribution...
percona/pmm                  	0.3.0        	2.29.0     	A Helm chart for Percona Monitoring and Managem...
percona/ps-db                	0.2.0        	0.2.0      	A Helm chart for installing Percona Server Data...
percona/ps-operator          	0.2.0        	0.2.0      	A Helm chart for Deploying the Percona Kubernet...
percona/psmdb-db             	1.12.3       	1.12.0     	A Helm chart for installing Percona Server Mong...
percona/psmdb-operator       	1.12.1       	1.12.0     	A Helm chart for Deploying the Percona Kubernet...
percona/pxc-db               	1.11.5       	1.11.0     	A Helm chart for installing Percona XtraDB Clus...
percona/pxc-operator         	1.11.1       	1.11.0     	A Helm chart for Deploying the Percona XtraDB C...
```

## 拉取 chart

```bash
# helm pull percona/pxc-operator -d /usr/local/k8s/mysql

# helm pull percona/pxc-db -d /usr/local/k8s/mysql
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/mysql/pxc-db-1.11.5.tgz -C /usr/local/k8s/mysql

# vim /usr/local/k8s/mysql/pxc-db/values.yaml
```

```yml
pxc:
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
  persistence:
    enabled: true
    storageClass: "nfs-client"

haproxy:
  enabled: true
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"

proxysql:
  enabled: false
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
  persistence:
    enabled: true
    storageClass: "nfs-client"

backup:
  storages:
    fs-pvc:
      volume:
        persistentVolumeClaim:
          storageClassName: nfs-client

secrets:
  passwords:
    root: root
    xtrabackup: root
    monitor: root
    clustercheck: root
    proxyadmin: root
    pmmserver: root
    operator: root
    replication: root
```

注: 

- 如果节点的数量充足，就可以不用设置 pod 的容忍度
- haproxy 和 proxysql 同时只能有一个生效

查询主节点的污点信息:

```bash
# kubectl get nodes
NAME                STATUS   ROLES           AGE   VERSION
centos-docker-163   Ready    control-plane   16d   v1.24.3
centos-docker-164   Ready    <none>          16d   v1.24.3
centos-docker-165   Ready    <none>          16d   v1.24.3

# kubectl describe node centos-docker-163 | grep -A 1 Taints
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
                    node-role.kubernetes.io/master:NoSchedule
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/mysql/pxc-db-1.11.5.tgz

# helm package /usr/local/k8s/mysql/pxc-db -d /usr/local/k8s/mysql

# tree /usr/local/k8s/mysql
/usr/local/k8s/mysql
├── pxc-db
│   ├── Chart.yaml
│   ├── crds
│   │   └── crd.yaml
│   ├── production-values.yaml
│   ├── README.md
│   ├── templates
│   │   ├── cluster-secret.yaml
│   │   ├── cluster-ssl-secret.yaml
│   │   ├── cluster.yaml
│   │   ├── _helpers.tpl
│   │   ├── NOTES.txt
│   │   └── s3-secret.yaml
│   └── values.yaml
├── pxc-db-1.11.5.tgz
├── pxc-operator
│   ├── Chart.yaml
│   ├── crds
│   │   └── crd.yaml
│   ├── LICENSE.txt
│   ├── README.md
│   ├── templates
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── namespace.yaml
│   │   ├── NOTES.txt
│   │   ├── role-binding.yaml
│   │   └── role.yaml
│   └── values.yaml
└── pxc-operator-1.11.1.tgz
```

## 部署

## 部署 pxc-operator

```bash
# helm install pxc-operator pxc-operator-1.11.1.tgz -n iot
NAME: pxc-operator
LAST DEPLOYED: Thu Aug  4 11:20:30 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. pxc-operator deployed.
  If you would like to deploy an pxc-cluster set cluster.enabled to true in values.yaml
  Check the pxc-operator logs
    export POD=$(kubectl get pods -l app.kubernetes.io/name=pxc-operator --namespace iot --output name)
    kubectl logs $POD --namespace=iot
```

查看资源:

```bash
# kubectl get deploy -n iot | grep pxc-operator
pxc-operator                      1/1     1            1           5m

# kubectl get rs -n iot | grep pxc-operator
pxc-operator-7f474bf7b9                      1         1         1       5m

# kubectl get pod -n iot -o wide | grep pxc-operator
pxc-operator-7f474bf7b9-dvn8l                      1/1     Running     0               122m   10.244.2.106   centos-docker-165   <none>           <none>
```

## 部署 pxc-db

```bash
# helm install pxc-db /usr/local/k8s/mysql/pxc-db-1.11.5.tgz -n iot
NAME: pxc-db
LAST DEPLOYED: Thu Aug  4 14:34:55 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. To get a MySQL prompt inside your new cluster you can run:

    ROOT_PASSWORD=`kubectl -n iot get secrets pxc-db -o jsonpath="{.data.root}" | base64 --decode`
    kubectl -n iot exec -ti \
      pxc-db-pxc-0 -- mysql -uroot -p"$ROOT_PASSWORD"

2. To connect an Application running in the same Kubernetes cluster you can connect with:
    ROOT_PASSWORD=`kubectl -n iot get secrets pxc-db -o jsonpath="{.data.root}" | base64 --decode`


$ kubectl run -i --tty --rm percona-client --image=percona --restart=Never \
  -- mysql -h pxc-db-proxysql.iot.svc.cluster.local -uroot -p"$ROOT_PASSWORD"
```

查看资源:

```bash
# kubectl get sts -n iot | grep pxc-db
pxc-db-haproxy   3/3     6m33s
pxc-db-pxc       3/3     6m33s

# kubectl get pod -n iot -o wide | grep pxc-db
pxc-db-haproxy-0                                   2/2     Running     0               4m36s   10.244.2.115   centos-docker-165   <none>           <none>
pxc-db-haproxy-1                                   2/2     Running     0               3m11s   10.244.1.138   centos-docker-164   <none>           <none>
pxc-db-haproxy-2                                   2/2     Running     0               2m50s   10.244.0.7     centos-docker-163   <none>           <none>
pxc-db-pxc-0                                       3/3     Running     0               4m36s   10.244.1.137   centos-docker-164   <none>           <none>
pxc-db-pxc-1                                       3/3     Running     0               3m13s   10.244.0.6     centos-docker-163   <none>           <none>
pxc-db-pxc-2                                       3/3     Running     0               2m10s   10.244.2.116   centos-docker-165   <none>           <none>

# kubectl get pvc,pv -n iot | grep pxc-db
persistentvolumeclaim/datadir-pxc-db-pxc-0         Bound    pvc-e6b0235a-05fe-49e1-8484-bcb4e8242dcf   8Gi        RWO            nfs-client     5m
persistentvolumeclaim/datadir-pxc-db-pxc-1         Bound    pvc-a6403b8f-7e29-4fe5-a119-8b27ac003fbf   8Gi        RWO            nfs-client     3m37s
persistentvolumeclaim/datadir-pxc-db-pxc-2         Bound    pvc-0c47091b-59a5-4c4a-b8e2-da47a5fc6a78   8Gi        RWO            nfs-client     2m34s
persistentvolume/pvc-0c47091b-59a5-4c4a-b8e2-da47a5fc6a78   8Gi        RWO            Delete           Bound    iot/datadir-pxc-db-pxc-2         nfs-client              2m34s
persistentvolume/pvc-a6403b8f-7e29-4fe5-a119-8b27ac003fbf   8Gi        RWO            Delete           Bound    iot/datadir-pxc-db-pxc-1         nfs-client              3m37s
persistentvolume/pvc-e6b0235a-05fe-49e1-8484-bcb4e8242dcf   8Gi        RWO            Delete           Bound    iot/datadir-pxc-db-pxc-0         nfs-client              5m

# kubectl get svc -n iot | grep pxc-db
pxc-db-haproxy                    ClusterIP   10.111.145.168   <none>        3306/TCP,3309/TCP,33062/TCP,33060/TCP   6m13s
pxc-db-haproxy-replicas           ClusterIP   10.104.135.178   <none>        3306/TCP                                6m13s
pxc-db-pxc                        ClusterIP   None             <none>        3306/TCP,33062/TCP,33060/TCP            6m13s
pxc-db-pxc-unready                ClusterIP   None             <none>        3306/TCP,33062/TCP,33060/TCP            6m13s
```

## 内部访问 mysql 集群

在 pxc-db-pxc-0 的 pod 里新建 ```test``` 数据库:

```bash
# kubectl exec -it pxc-db-pxc-0  -n iot -- /bin/sh

# mysql

mysql> create database test;

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
```

查看 pxc-db-pxc-1 的 pod 里的数据库同步情况:

```bash
# kubectl exec -it pxc-db-pxc-1 -n iot -- /bin/sh

# mysql

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
```

查看 pxc-db-pxc-2 的 pod 里的数据库同步情况:

```bash
# kubectl exec -it pxc-db-pxc-2 -n iot -- /bin/sh

# mysql

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
```

## 外部访问 mysql 集群

创建 NodePort 类型的 Service:

```bash
# vim /usr/local/k8s/mysql/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: pxc-db-pxc-service
  namespace: iot
spec:
  selector:
    app.kubernetes.io/component: pxc
    app.kubernetes.io/instance: pxc-db
    app.kubernetes.io/name: percona-xtradb-cluster
  ports:
    - name: mysql
      protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 30306
    - name: mysql-admin
      protocol: TCP
      port: 33062
      targetPort: 33062
      nodePort: 30307
    - name: mysqlx
      protocol: TCP
      port: 33060
      targetPort: 33060
      nodePort: 30308
  type: NodePort
```

```bash
# kubectl apply -f /usr/local/k8s/mysql/service.yaml

# kubectl get svc pxc-db-pxc-service -n iot
NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                                          AGE
pxc-db-pxc-service   NodePort   10.104.103.158   <none>        3306:30306/TCP,33062:30307/TCP,33060:30308/TCP   15s
```

外部服务器连接 mysql 集群:

```bash
# mysql -uroot -proot -h192.168.5.163 -P30306

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
```
