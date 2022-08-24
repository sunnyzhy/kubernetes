# helm - minio 集群

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

- 集群相关资料:

    ```
    http://docs.minio.org.cn/docs/master/deploy-minio-on-kubernetes
    ```

- minio 账号的密码长度必须是 ```>= 8```

## 创建 minio 目录

```bash
# mkdir -p /usr/local/k8s/minio
```

## 查找 chart

```bash
# helm search repo minio
NAME         	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/minio 	0.5.5        	           	Distributed object storage server built for clo...
bitnami/minio	11.8.1       	2022.8.11  	MinIO(R) is an object storage server, compatibl...
```

## 拉取 chart

```bash
# helm pull bitnami/minio -d /usr/local/k8s/minio
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/minio/minio-11.8.1.tgz -C /usr/local/k8s/minio

# vim /usr/local/k8s/minio/minio/values.yaml
```

```yml
global:
  storageClass: "nfs-client"

mode: distributed

auth:
  rootUser: admin
  rootPassword: "password"

extraEnvVars:
  - name: TZ
    value: Asia/Shanghai

service:
  nodePorts:
    api: "30900"
    console: "30901"

ingress:
  enabled: true
  ingressClassName: "nginx"
  hostname: iot.minio
```

### 部署对外的 service

在 ```service.yaml``` 里添加 NodePort 类型的 Service，对外提供服务（非必需，功能同 ingress）:

```bash
# vim /usr/local/k8s/minio/minio/templates/service.yaml
```

```yml

---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "common.names.fullname" . }}-service
  namespace: {{ .Release.Namespace | quote }}
  labels: {{- include "common.labels.standard" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
spec:
  type: NodePort
  ports:
    - name: minio-api
      port: {{ .Values.service.ports.api }}
      targetPort: minio-api
      nodePort: {{ .Values.service.nodePorts.api }}
    - name: minio-console
      port: {{ .Values.service.ports.console }}
      targetPort: minio-console
      nodePort: {{ .Values.service.nodePorts.console }}
  selector: {{- include "common.labels.matchLabels" . | nindent 4 }}
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/minio/minio-11.8.1.tgz

# helm package /usr/local/k8s/minio/minio -d /usr/local/k8s/minio

# tree /usr/local/k8s/minio
/usr/local/k8s/minio
├── minio
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
│   │   ├── api-ingress.yaml
│   │   ├── distributed
│   │   │   ├── headless-svc.yaml
│   │   │   ├── pdb.yaml
│   │   │   └── statefulset.yaml
│   │   ├── extra-list.yaml
│   │   ├── gateway
│   │   │   ├── deployment.yaml
│   │   │   ├── hpa.yaml
│   │   │   └── pdb.yaml
│   │   ├── _helpers.tpl
│   │   ├── ingress.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── NOTES.txt
│   │   ├── prometheusrule.yaml
│   │   ├── provisioning-configmap.yaml
│   │   ├── provisioning-job.yaml
│   │   ├── pvc.yaml
│   │   ├── secrets.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── servicemonitor.yaml
│   │   ├── service.yaml
│   │   ├── standalone
│   │   │   └── deployment.yaml
│   │   └── tls-secrets.yaml
│   └── values.yaml
└── minio-11.8.1.tgz
```

## 部署

```bash
# helm install minio-cluster /usr/local/k8s/minio/minio-11.8.1.tgz -n iot
NAME: minio-cluster
LAST DEPLOYED: Mon Aug 15 11:49:43 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: minio
CHART VERSION: 11.8.1
APP VERSION: 2022.8.11

** Please be patient while the chart is being deployed **

MinIO&reg; can be accessed via port  on the following DNS name from within your cluster:

   minio-cluster.iot.svc.cluster.local

To get your credentials run:

   export ROOT_USER=$(kubectl get secret --namespace iot minio-cluster -o jsonpath="{.data.root-user}" | base64 -d)
   export ROOT_PASSWORD=$(kubectl get secret --namespace iot minio-cluster -o jsonpath="{.data.root-password}" | base64 -d)

To connect to your MinIO&reg; server using a client:

- Run a MinIO&reg; Client pod and append the desired command (e.g. 'admin info'):

   kubectl run --namespace iot minio-cluster-client \
     --rm --tty -i --restart='Never' \
     --env MINIO_SERVER_ROOT_USER=$ROOT_USER \
     --env MINIO_SERVER_ROOT_PASSWORD=$ROOT_PASSWORD \
     --env MINIO_SERVER_HOST=minio-cluster \
     --image docker.io/bitnami/minio-client:2022.8.5-debian-11-r2 -- admin info minio

To access the MinIO&reg; web UI:

- Get the MinIO&reg; URL:

   echo "MinIO&reg; web URL: http://127.0.0.1:9001/minio"
   kubectl port-forward --namespace iot svc/minio-cluster 9001:9001
```

查看资源:

```bash
# kubectl get sts -n iot | grep minio
minio-cluster                        4/4     3m52s

# kubectl get pod -n iot -o wide | grep minio
minio-cluster-0                                    1/1     Running            1                  4m7s   10.244.1.151   centos-docker-164   <none>           <none>
minio-cluster-1                                    1/1     Running            1 (2m48s ago)      4m7s   10.244.2.119   centos-docker-165   <none>           <none>
minio-cluster-2                                    1/1     Running            1 (3m12s ago)      4m7s   10.244.1.152   centos-docker-164   <none>           <none>
minio-cluster-3                                    1/1     Running            3 (40s ago)        4m7s   10.244.2.120   centos-docker-165   <none>           <none>

# kubectl get pvc,pv -n iot | grep minio
persistentvolumeclaim/data-minio-cluster-0                    Bound    pvc-947863a9-cb26-4d80-a9eb-a7489b33c9c3   8Gi        RWO            nfs-client     4m15s
persistentvolumeclaim/data-minio-cluster-1                    Bound    pvc-0db60e5f-829d-4d83-9d0f-16d56723d197   8Gi        RWO            nfs-client     4m15s
persistentvolumeclaim/data-minio-cluster-2                    Bound    pvc-c96b141a-935a-46b8-8412-b28a05d46dc3   8Gi        RWO            nfs-client     4m15s
persistentvolumeclaim/data-minio-cluster-3                    Bound    pvc-d6ef1999-234f-4f01-9e51-d8c8c438c2a5   8Gi        RWO            nfs-client     4m15s
persistentvolume/pvc-0db60e5f-829d-4d83-9d0f-16d56723d197   8Gi        RWO            Delete           Bound    iot/data-minio-cluster-1                    nfs-client              4m15s
persistentvolume/pvc-947863a9-cb26-4d80-a9eb-a7489b33c9c3   8Gi        RWO            Delete           Bound    iot/data-minio-cluster-0                    nfs-client              4m15s
persistentvolume/pvc-c96b141a-935a-46b8-8412-b28a05d46dc3   8Gi        RWO            Delete           Bound    iot/data-minio-cluster-2                    nfs-client              4m15s
persistentvolume/pvc-d6ef1999-234f-4f01-9e51-d8c8c438c2a5   8Gi        RWO            Delete           Bound    iot/data-minio-cluster-3                    nfs-client              4m15s

# kubectl get svc -n iot | grep minio
minio-cluster                           ClusterIP   10.96.108.121    <none>        9000/TCP,9001/TCP                                                                            4m45s
minio-cluster-headless                  ClusterIP   None             <none>        9000/TCP,9001/TCP                                                                            4m45s

# kubectl get ingress -n iot | grep minio
minio-cluster            nginx   iot.minio                  80      4m56s
```

## 内部访问 minio 集群

略

## 外部访问 minio 集群

### 通过对外 Service 的方式

在浏览器的地址栏里输入:```http://192.168.5.163:30901/```; ```用户名/密码```: ```admin/password```

### 通过 ingress 的方式

在任一外部服务器的 hosts 文件里配置域名映射:

```bash
# vim /etc/hosts
192.168.5.165 iot.minio
```

在浏览器的地址栏里输入:```http://iot.minio/```, ```用户名/密码: admin/password```
