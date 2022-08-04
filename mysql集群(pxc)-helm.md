# mysql 集群 (pxc) - helm

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
# helm pull aliyun/percona-xtradb-cluster -d /usr/local/k8s/mysql
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/mysql/percona-xtradb-cluster-0.0.2.tgz -C /usr/local/k8s/mysql

# vim /usr/local/k8s/mysql/percona-xtradb-cluster/values.yaml
```

```yml
 mysqlRootPassword: root
 
 xtraBackupPassword: root
 
 persistence:
  enabled: true
  storageClass: "nfs-client"
```

```bash
# sed -i -e 's+apps/v1beta2+apps/v1+' -e 's+busybox:1.25.0+busybox:1.25+' /usr/local/k8s/mysql/percona-xtradb-cluster/templates/statefulset.yaml
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/mysql/percona-xtradb-cluster-0.0.2.tgz

# helm package /usr/local/k8s/mysql/percona-xtradb-cluster

# tree /usr/local/k8s/mysql
/usr/local/k8s/mysql
├── percona-xtradb-cluster
│   ├── Chart.yaml
│   ├── files
│   │   ├── entrypoint.sh
│   │   ├── functions.sh
│   │   └── node.cnf
│   ├── README.md
│   ├── templates
│   │   ├── config-map_mysql-config.yaml
│   │   ├── config-map_startup-scripts.yaml
│   │   ├── _helpers.tpl
│   │   ├── NOTES.txt
│   │   ├── secrets.yaml
│   │   ├── service-metrics.yaml
│   │   ├── service-percona.yaml
│   │   ├── service-repl.yaml
│   │   └── statefulset.yaml
│   └── values.yaml
└── percona-xtradb-cluster-0.0.2.tgz
```

## 部署

```bash
# helm install mysql-pxc /usr/local/k8s/mysql/percona-xtradb-cluster-0.0.2.tgz -n iot
NAME: mysql-pxc
LAST DEPLOYED: Wed Aug  3 15:22:08 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Percona can be accessed via port 3306 on the following DNS name from within your cluster:
mysql-pxc-pxc.iot.svc.cluster.local

To get your root password run (this password can only be used from inside the container):

    $ kubectl get secret --namespace iot mysql-pxc-pxc -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo

To get your xtradb backup password run:

    $ kubectl get secret --namespace iot mysql-pxc-pxc -o jsonpath="{.data.xtrabackup-password}" | base64 --decode; echo

To check the size of the xtradb cluster:

    $ kubectl exec --namespace iot -ti mysql-pxc-pxc-0 -c database -- mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep_cluster_size'"

To connect to your database:

1. Run a command in the first pod in the StatefulSet:

    $ kubectl exec --namespace iot -ti mysql-pxc-pxc-0 -c database -- mysql
To view your Percona XtraDB Cluster logs run:

  $ kubectl logs -f mysql-pxc-pxc-0 logs
```

查看资源:

```bash
# kubectl get sts -n iot | grep mysql-pxc
mysql-pxc-pxc             3/3     4m56s

# kubectl get pod -n iot -o wide | grep mysql-pxc
mysql-pxc-pxc-0                                    2/2     Running     0               5m10s   10.244.2.102   centos-docker-165   <none>           <none>
mysql-pxc-pxc-1                                    2/2     Running     0               3m29s   10.244.1.121   centos-docker-164   <none>           <none>
mysql-pxc-pxc-2                                    2/2     Running     0               2m7s    10.244.1.122   centos-docker-164   <none>           <none>

# kubectl get pvc,pv -n iot | grep mysql-pxc
persistentvolumeclaim/mysql-data-mysql-pxc-0           Bound    pvc-7287635b-6db3-45ec-baab-26e21c1e08ce   8Gi        RWO            nfs-client     20m
persistentvolumeclaim/mysql-data-mysql-pxc-pxc-0       Bound    pvc-362db75e-8797-45af-92ee-87f655bc7937   8Gi        RWO            nfs-client     5m35s
persistentvolumeclaim/mysql-data-mysql-pxc-pxc-1       Bound    pvc-5da7a387-d6fc-47a4-81cf-02e8aa550652   8Gi        RWO            nfs-client     3m54s
persistentvolumeclaim/mysql-data-mysql-pxc-pxc-2       Bound    pvc-29a650ec-d069-416f-863c-ae9ac6555154   8Gi        RWO            nfs-client     2m32s
persistentvolume/pvc-29a650ec-d069-416f-863c-ae9ac6555154   8Gi        RWO            Delete           Bound    iot/mysql-data-mysql-pxc-pxc-2       nfs-client              2m32s
persistentvolume/pvc-362db75e-8797-45af-92ee-87f655bc7937   8Gi        RWO            Delete           Bound    iot/mysql-data-mysql-pxc-pxc-0       nfs-client              5m35s
persistentvolume/pvc-5da7a387-d6fc-47a4-81cf-02e8aa550652   8Gi        RWO            Delete           Bound    iot/mysql-data-mysql-pxc-pxc-1       nfs-client              3m54s
persistentvolume/pvc-7287635b-6db3-45ec-baab-26e21c1e08ce   8Gi        RWO            Delete           Bound    iot/mysql-data-mysql-pxc-0           nfs-client              20m

# kubectl get svc -n iot | grep mysql-pxc
mysql-pxc-pxc                      ClusterIP   10.110.242.195   <none>        3306/TCP                     6m6s
mysql-pxc-pxc-repl                 ClusterIP   None             <none>        4567/TCP,4568/TCP,4444/TCP   6m6s
```

## 内部访问 mysql 集群

在 mysql-pxc-pxc-0 的 pod 里新建 ```test``` 数据库:

```bash
# kubectl exec -it mysql-pxc-pxc-0 -n iot -- /bin/sh

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

查看 mysql-pxc-pxc-1 的 pod 里的数据库同步情况:

```bash
# kubectl exec -it mysql-pxc-pxc-1 -n iot -- /bin/sh

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

查看 mysql-pxc-pxc-2 的 pod 里的数据库同步情况:

```bash
# kubectl exec -it mysql-pxc-pxc-2 -n iot -- /bin/sh

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

进入 mysql-pxc-pxc-0 的 pod, 修改 root 账号的 host:

```bash
# kubectl exec -it mysql-pxc-pxc-0 -n iot -- /bin/sh

# mysql 

mysql> use mysql;

mysql> select user,host from user where user='root' \G;
*************************** 1. row ***************************
user: root
host: localhost
*************************** 2. row ***************************
user: root
host: 127.0.0.1

mysql> update user set user.Host='%' where user.User='root' and user.Host='localhost';

mysql> select user,host from user where user='root' \G;
*************************** 1. row ***************************
user: root
host: %
*************************** 2. row ***************************
user: root
host: 127.0.0.1

mysql> flush privileges;
```

进入 mysql-pxc-pxc-1 的 pod, 修改 root 账号的 host, 方法同上。

进入 mysql-pxc-pxc-2 的 pod, 修改 root 账号的 host, 方法同上。

创建 NodePort 类型的 Service:

```bash
# vim /usr/local/k8s/mysql/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: mysql-pxc-pxc-service
  namespace: iot
spec:
  selector:
    app: mysql-pxc-pxc
    release: mysql-pxc
  ports:
    - name: mysql
      protocol: TCP
      port: 3306
      targetPort: mysql
      nodePort: 30306
  type: NodePort
```

```bash
# kubectl apply -f /usr/local/k8s/mysql/service.yaml

# kubectl get svc mysql-pxc-pxc-service -n iot
NAME                    TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
mysql-pxc-pxc-service   NodePort   10.109.2.4   <none>        3306:30306/TCP   51s
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
