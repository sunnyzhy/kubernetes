# 安装 helm

[helm](https://github.com/helm/helm/releases 'helm')

## 安装 helm

```bash
# wget -c https://get.helm.sh/helm-v3.9.1-linux-amd64.tar.gz -P /usr/local/

# tar Cxzvf /usr/local /usr/local/helm-v3.9.1-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md

# mv /usr/local/linux-amd64/helm /usr/local/bin/

# rm -rf /usr/local/linux-amd64
```

## 配置 chart 仓库

### 添加阿里云的 chart 仓库

```bash
# helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

### 添加 bitnami 的 chart 仓库

```bash
# helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 更新 chart 仓库

```bash
# helm repo update

# helm repo list
NAME   	URL                                                   
aliyun 	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
bitnami	https://charts.bitnami.com/bitnami   
```

### 删除 chart 仓库

```bash
# helm repo remove <NAME>
```

## 查找 chart

查找 redis chart:

```bash
# helm search repo redis
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/redis         	1.1.15       	4.0.8      	Open source, advanced key-value store. It is of...
aliyun/redis-ha      	2.0.1        	           	Highly available Redis cluster with multiple se...
bitnami/redis        	17.0.6       	7.0.4      	Redis(R) is an open source, advanced key-value ...
bitnami/redis-cluster	8.1.2        	7.0.4      	Redis(R) is an open source, scalable, distribut...
aliyun/sensu         	0.2.0        	           	Sensu monitoring framework backed by the Redis ...
```

查找 mysql chart:

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

## 卸载 release

```bash
# helm list -n <NAMESPACE>

# helm uninstall <NAME> -n <NAMESPACE>
```

## 自定义 chart

```bash
# mkdir -p /usr/local/k8s/my-chart

# helm create /usr/local/k8s/my-chart/nginx

# tree /usr/local/k8s/my-chart/nginx
/usr/local/k8s/my-chart/nginx
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 10 files
```
