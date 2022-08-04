# mysql 集群 (ms) - helm

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

## 查找 chart

```bash
# helm search repo mysql
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/mysql                 	0.3.5        	           	Fast, reliable, scalable, and easy to use open-...
bitnami/mysql                	9.2.3        	8.0.30     	MySQL is a fast, reliable, scalable, and easy t...
aliyun/percona               	0.3.0        	           	free, fully compatible, enhanced, open source d...
aliyun/percona-xtradb-cluster	0.0.2        	5.7.19     	free, fully compatible, enhanced, open source d...
bitnami/phpmyadmin           	10.2.1       	5.2.0      	phpMyAdmin is a free software tool written in P...
aliyun/gcloud-sqlproxy       	0.2.3        	           	Google Cloud SQL Proxy                            
aliyun/mariadb               	2.1.6        	10.1.31    	Fast, reliable, scalable, and easy to use open-...
bitnami/mariadb              	11.1.2       	10.6.8     	MariaDB is an open source, community-developed ...
bitnami/mariadb-galera       	7.3.7        	10.6.8     	MariaDB Galera is a multi-primary database clus...
```

## 拉取 chart

```bash
# helm pull bitnami/mysql -d /usr/local/k8s/mysql
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/mysql/mysql-9.2.3.tgz -C /usr/local/k8s/mysql

# vim /usr/local/k8s/mysql/mysql/values.yaml
```

```yml
global:
  storageClass: "nfs-client"

architecture: replication

auth:
  rootPassword: "root"
```

参数说明:

- storageClass: 使用 NFS 存储
- auth.rootPassword: root 账号的密码
- architecture: 默认是 ```standalone```，取值范围: ```standalone/replication```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/mysql/mysql-9.2.3.tgz

# helm package /usr/local/k8s/mysql/mysql

# tree /usr/local/k8s/mysql
/usr/local/k8s/mysql
├── mysql
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
│   │   ├── extra-list.yaml
│   │   ├── _helpers.tpl
│   │   ├── metrics-svc.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── NOTES.txt
│   │   ├── primary
│   │   │   ├── configmap.yaml
│   │   │   ├── initialization-configmap.yaml
│   │   │   ├── pdb.yaml
│   │   │   ├── statefulset.yaml
│   │   │   ├── svc-headless.yaml
│   │   │   └── svc.yaml
│   │   ├── prometheusrule.yaml
│   │   ├── rolebinding.yaml
│   │   ├── role.yaml
│   │   ├── secondary
│   │   │   ├── configmap.yaml
│   │   │   ├── pdb.yaml
│   │   │   ├── statefulset.yaml
│   │   │   ├── svc-headless.yaml
│   │   │   └── svc.yaml
│   │   ├── secrets.yaml
│   │   ├── serviceaccount.yaml
│   │   └── servicemonitor.yaml
│   ├── values.schema.json
│   └── values.yaml
└── mysql-9.2.3.tgz
```

## 部署

```bash
# helm install mysql-cluster /usr/local/k8s/mysql/mysql-9.2.3.tgz -n iot
NAME: mysql-cluster
LAST DEPLOYED: Wed Aug  3 09:10:09 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.2.3
APP VERSION: 8.0.30

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace iot

Services:

  echo Primary: mysql-cluster-primary.iot.svc.cluster.local:3306
  echo Secondary: mysql-cluster-secondary.iot.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace iot mysql-cluster -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-cluster-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.30-debian-11-r1 --namespace iot --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql-cluster-primary.iot.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"

  3. To connect to secondary service (read-only):

      mysql -h mysql-cluster-secondary.iot.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

查看资源:

```bash
# kubectl get sts -n iot | grep mysql
mysql-cluster-primary     1/1     2m29s
mysql-cluster-secondary   1/1     2m29s

# kubectl get pod -n iot -o wide | grep mysql
mysql-cluster-primary-0                            1/1     Running     0               2m43s   10.244.2.101   centos-docker-165   <none>           <none>
mysql-cluster-secondary-0                          1/1     Running     0               2m43s   10.244.1.119   centos-docker-164   <none>           <none>

# kubectl get pvc,pv -n iot | grep mysql
persistentvolumeclaim/data-mysql-cluster-primary-0     Bound    pvc-6bef4a71-a754-42e1-949b-e0abc1659e34   8Gi        RWO            nfs-client     3m12s
persistentvolumeclaim/data-mysql-cluster-secondary-0   Bound    pvc-052746cd-bb6a-4c99-8115-0229a647e8b4   8Gi        RWO            nfs-client     3m12s
persistentvolume/pvc-052746cd-bb6a-4c99-8115-0229a647e8b4   8Gi        RWO            Delete           Bound    iot/data-mysql-cluster-secondary-0   nfs-client              3m11s
persistentvolume/pvc-6bef4a71-a754-42e1-949b-e0abc1659e34   8Gi        RWO            Delete           Bound    iot/data-mysql-cluster-primary-0     nfs-client              3m11s

# kubectl get svc -n iot | grep mysql
mysql-cluster-primary              ClusterIP   10.97.76.242     <none>        3306/TCP             3m49s
mysql-cluster-primary-headless     ClusterIP   None             <none>        3306/TCP             3m49s
mysql-cluster-secondary            ClusterIP   10.107.219.105   <none>        3306/TCP             3m49s
mysql-cluster-secondary-headless   ClusterIP   None             <none>        3306/TCP             3m49s
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
  name: mysql-cluster-primary-service
  namespace: iot
spec:
  selector:
    app.kubernetes.io/component: primary
    app.kubernetes.io/instance: mysql-cluster
    app.kubernetes.io/name: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: mysql
      nodePort: 30306
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-cluster-secondary-service
  namespace: iot
spec:
  selector:
    app.kubernetes.io/component: secondary
    app.kubernetes.io/instance: mysql-cluster
    app.kubernetes.io/name: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: mysql
      nodePort: 30307
  type: NodePort
```

```bash
# kubectl apply -f /usr/local/k8s/mysql/service.yaml

# kubectl get svc mysql-cluster-primary-service mysql-cluster-secondary-service -n iot
NAME                              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mysql-cluster-primary-service     NodePort   10.110.165.179   <none>        3306:30306/TCP   63s
mysql-cluster-secondary-service   NodePort   10.103.168.40    <none>        3306:30307/TCP   63s
```

外部服务器连接 mysql-primary:

```bash
# mysql -uroot -proot -h192.168.5.163 -P30306

mysql> create database test;

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database        |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
```

外部服务器连接 mysql-secondary:

```bash
# mysql -uroot -proot -h192.168.5.163 -P30307

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| my_database        |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
```
