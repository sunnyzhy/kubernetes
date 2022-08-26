# helm - influxdb2

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

- influxdb2 账号的密码长度必须 ```>= 8```

- ***```influxdb 2.x``` 认证使用的是 ```token``` 而不是 ```用户名密码```***

## 创建 influxdb 目录

```bash
# mkdir -p /usr/local/k8s/influxdb
```

## 查找 chart

```bash
# helm search repo influxdb
NAME            	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/influxdb 	0.8.2        	           	Scalable datastore for metrics, events, and rea...
bitnami/influxdb	5.3.11       	2.3.0      	InfluxDB(TM) is an open source time-series data...
aliyun/kapacitor	0.5.0        	           	InfluxDB's native data processing engine. It ca...
bitnami/grafana 	7.6.5        	8.3.4      	Grafana is an open source, feature rich metrics...
```

## 拉取 chart

```bash
# helm pull bitnami/influxdb -d /usr/local/k8s/influxdb
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/influxdb/influxdb-5.3.11.tgz -C /usr/local/k8s/influxdb
```

### 修改 influxdb 配置

```bash
# vim /usr/local/k8s/influxdb/influxdb/values.yaml
```

```yml
auth:
  admin:
    password: "password"

influxdb:
  service:
    nodePorts:
      http: "30186"
      rpc: "30188"

persistence:
  storageClass: "nfs-client"

ingress:
  enabled: true
  hostname: iot.influxdb
  ingressClassName: "nginx"
```

#### 部署 service 实例

在 ```service.yaml``` 里添加 NodePort 类型的 Service，对外提供服务（非必需，功能同 ingress）:

```bash
# vim /usr/local/k8s/influxdb/influxdb/templates/service.yaml
```

```yml

---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "common.names.fullname" . }}-service
  namespace: {{ .Release.Namespace | quote }}
  labels: {{- include "common.labels.standard" . | nindent 4 }}
    app.kubernetes.io/component: influxdb
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
spec:
  type: NodePort
  ports:
    - port: {{ coalesce .Values.influxdb.service.ports.http .Values.influxdb.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      nodePort: {{ .Values.influxdb.service.nodePorts.http }}
    - port: {{ coalesce .Values.influxdb.service.ports.rpc .Values.influxdb.service.rpcPort }}
      targetPort: rpc
      protocol: TCP
      name: rpc
      nodePort: {{ .Values.influxdb.service.nodePorts.rpc }}
  selector: {{- include "common.labels.matchLabels" . | nindent 4 }}
    app.kubernetes.io/component: influxdb
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/influxdb/influxdb-5.3.11.tgz

# helm package /usr/local/k8s/influxdb/influxdb -d /usr/local/k8s/influxdb

# tree /usr/local/k8s/influxdb
/usr/local/k8s/influxdb
├── influxdb
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
│   ├── files
│   │   ├── conf
│   │   │   └── README.md
│   │   └── docker-entrypoint-initdb.d
│   │       └── README.md
│   ├── README.md
│   ├── templates
│   │   ├── configmap-backup.yaml
│   │   ├── configmap-initdb-scripts.yaml
│   │   ├── configmap.yaml
│   │   ├── cronjob-backup.yaml
│   │   ├── deployment.yaml
│   │   ├── extra-list.yaml
│   │   ├── _helpers.tpl
│   │   ├── ingress.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── NOTES.txt
│   │   ├── podsecuritypolicy.yaml
│   │   ├── pvc-backup.yaml
│   │   ├── pvc.yaml
│   │   ├── rolebinding.yaml
│   │   ├── role.yaml
│   │   ├── secrets-backup.yaml
│   │   ├── secrets.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── service-collectd.yaml
│   │   ├── service-metrics.yaml
│   │   ├── servicemonitor.yaml
│   │   └── service.yaml
│   └── values.yaml
└── influxdb-5.3.11.tgz
```

## 部署

```bash
# helm install influxdb /usr/local/k8s/influxdb/influxdb-5.3.11.tgz -n iot
NAME: influxdb
LAST DEPLOYED: Thu Aug 25 18:57:04 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: influxdb
CHART VERSION: 5.3.11
APP VERSION: 2.3.0

** Please be patient while the chart is being deployed **

InfluxDB&trade; can be accessed through following DNS names from within your cluster:

    InfluxDB&trade;: influxdb.iot.svc.cluster.local (port 8086)

To connect to your database run the following commands:

    kubectl run influxdb-client --rm --tty -i --restart='Never' --namespace iot  \
        --image docker.io/bitnami/influxdb:2.3.0-debian-11-r17 \
        --command -- influx -host influxdb -port 8086

To connect to your database from outside the cluster execute the following commands:

  e.g.:

     influx -host iot.influxdb -port 80
```

查看资源:

```bash
# kubectl get deploy -n iot | grep influxdb
influxdb                          1/1     1            1           2m16s

# kubectl get rs -n iot | grep influxdb
influxdb-6cc97cc4df                          1         1         1       2m29s

# kubectl get pod -n iot -o wide | grep influxdb
influxdb-6cc97cc4df-92zrb                          1/1     Running   0             2m39s   10.244.2.60   centos-docker-165   <none>           <none>

# kubectl get pvc,pv -n iot | grep influxdb
persistentvolumeclaim/influxdb                                      Bound    pvc-9419832d-c6c5-4055-8721-b7586cea4c4f   8Gi        RWO            nfs-client     2m57s
persistentvolume/pvc-9419832d-c6c5-4055-8721-b7586cea4c4f   8Gi        RWO            Delete           Bound    iot/influxdb                                      nfs-client              2m57s

# kubectl get svc -n iot | grep influxdb
influxdb                                ClusterIP   10.110.240.60    <none>        8086/TCP,8088/TCP                                                             3m15s
influxdb-service                        NodePort    10.106.225.99    <none>        8086:30186/TCP,8088:30188/TCP                                                 3m15s

# kubectl get ingress -n iot | grep influxdb
influxdb                       nginx   iot.influxdb   10.108.27.226   80      3m29s
```

## 获取 token

### 从 pod 的 configs 里获取

进入 pod:

```bash
# kubectl exec -it influxdb-6cc97cc4df-92zrb -n iot -- /bin/sh
```

查看 configs:

```bash
$ cat /bitnami/influxdb/configs | grep token
  token = "hhzTOUNBLOkf2kn2TbhN"
#   token = "XXX"
#   token = "XXX"
#   token = "XXX"
```

```hhzTOUNBLOkf2kn2TbhN``` 即为 admin 账号的 token

### 从 secret 里获取

```bash
# echo $(kubectl get secret influxdb -n iot -o jsonpath="{.data.admin-user-token}" | base64 -d)
hhzTOUNBLOkf2kn2TbhN
```

### 从 UI 里获取

1. 在浏览器的地址栏里输入:```http://iot.influxdb/```, ```用户名/密码: admin/password```
2. 打开 ```API Tokens``` 页面
3. 在右侧列表里选择 ```admin's Token```，即可在弹出的页面里看到 token: ```hhzTOUNBLOkf2kn2TbhN```

## 内部访问 influxdb

进入 influxdb 的 pod :

```bash
# kubectl exec -it influxdb-6cc97cc4df-92zrb -n iot -- /bin/sh

$ influx bucket list -o "primary" --token hhzTOUNBLOkf2kn2TbhN
ID			Name		Retention	Shard group duration	Organization ID		Schema Type
51e0c9235583dd7e	_monitoring	168h0m0s	24h0m0s			0b690b959a9196b2	implicit
e6b3655fb5d30298	_tasks		72h0m0s		24h0m0s			0b690b959a9196b2	implicit
c5689c856aab7b05	primary		infinite	168h0m0s		0b690b959a9196b2	implicit
```

## 外部访问 influxdb

### 通过对外 Service 的方式

在浏览器的地址栏里输入:```http://192.168.5.163:30186/```, ```用户名/密码: admin/password```

### 通过 ingress 的方式

在任一外部服务器的 hosts 文件里配置域名映射:

```bash
# vim /etc/hosts
192.168.5.165 iot.influxdb
```

在浏览器的地址栏里输入:```http://iot.influxdb/```, ```用户名/密码: admin/password```

***注: API Tokens 包含当前用户的 Token，与后端整合时需要用到。***
