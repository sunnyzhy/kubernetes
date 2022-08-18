# helm - rocketmq 集群

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
    https://github.com/apache/rocketmq-operator

    https://github.com/itboon/rocketmq-helm
    ```

- rocketmq 集群为 ```双主双从 + 异步模式```

## 创建 rocketmq 目录

```bash
# mkdir -p /usr/local/k8s/rocketmq

# helm create /usr/local/k8s/rocketmq/rocketmq

# tree /usr/local/k8s/rocketmq/rocketmq
/usr/local/k8s/rocketmq/rocketmq
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
```

## 修改配置

### 修改 Chart.yaml

```bash
# vim /usr/local/k8s/rocketmq/rocketmq/Chart.yaml
```

```yml
version: 4.9.4

appVersion: '4.9.4'
```

### 修改 values.yaml

```bash
# vim /usr/local/k8s/rocketmq/rocketmq/values.yaml
```

```yml
image:
  repository: apache/rocketmq
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ''

persistence:
  storageClass: nfs-client
  accessMode: ReadWriteMany
  size: 10Gi

namesrv:
  replicas: 2

dashboard:
  image:
    repository: apacherocketmq/rocketmq-dashboard
    pullPolicy: IfNotPresent
    tag: '1.0.0'
```

### 部署配置文件存储卷

```bash
# vim /usr/local/k8s/rocketmq/rocketmq/templates/configmap.yaml
```

```yml
{{- $fullname := include "rocketmq.fullname" . -}}
{{- $namesrvReplicas := int .Values.namesrv.replicas }}

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rocketmq-config
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
data:
  'broker-a.properties': |+
    brokerClusterName=RocketmqCluster
    brokerName=broker-a
    brokerId=0
    deleteWhen=04
    fileReservedTime=48
    brokerRole=ASYNC_MASTER
    flushDiskType=ASYNC_FLUSH
    namesrvAddr={{ range $i := until $namesrvReplicas }}{{ if gt $i 0 }}{{ printf ";" }}{{ end }}{{ printf "%s-namesrv-%d.%s-namesrv-hl:9876" $fullname $i $fullname }}{{ end }}
  'broker-a-s.properties': |+
    brokerClusterName=RocketmqCluster
    brokerName=broker-a-s
    brokerId=1
    deleteWhen=04
    fileReservedTime=48
    brokerRole=SLAVE
    flushDiskType=ASYNC_FLUSH
    namesrvAddr={{ range $i := until $namesrvReplicas }}{{ if gt $i 0 }}{{ printf ";" }}{{ end }}{{ printf "%s-namesrv-%d.%s-namesrv-hl:9876" $fullname $i $fullname }}{{ end }}
  'broker-b.properties': |+
    brokerClusterName=RocketmqCluster
    brokerName=broker-b
    brokerId=0
    deleteWhen=04
    fileReservedTime=48
    brokerRole=ASYNC_MASTER
    flushDiskType=ASYNC_FLUSH
    namesrvAddr={{ range $i := until $namesrvReplicas }}{{ if gt $i 0 }}{{ printf ";" }}{{ end }}{{ printf "%s-namesrv-%d.%s-namesrv-hl:9876" $fullname $i $fullname }}{{ end }}
  'broker-b-s.properties': |+
    brokerClusterName=RocketmqCluster
    brokerName=broker-b-s
    brokerId=1
    deleteWhen=04
    fileReservedTime=48
    brokerRole=SLAVE
    flushDiskType=ASYNC_FLUSH
    namesrvAddr={{ range $i := until $namesrvReplicas }}{{ if gt $i 0 }}{{ printf ";" }}{{ end }}{{ printf "%s-namesrv-%d.%s-namesrv-hl:9876" $fullname $i $fullname }}{{ end }}
```

### 部署 rocketmq 容器实例

```bash
# vim /usr/local/k8s/rocketmq/rocketmq/templates/statefulset.yaml
```

```yml
{{- $rocketmqVersion := .Values.image.tag | default .Chart.AppVersion }}

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "rocketmq.fullname" . }}-namesrv
  namespace: {{ .Release.Namespace }}
  labels:
    app: namesrv
    app.kubernetes.io/component: namesrv
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  serviceName: {{ include "rocketmq.fullname" . }}-namesrv-hl
  replicas: {{ .Values.namesrv.replicas  }}
  selector:
    matchLabels:
      app: namesrv
      app.kubernetes.io/component: namesrv
      app.kubernetes.io/name: {{ include "rocketmq.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
     labels:
       app: namesrv
       app.kubernetes.io/component: namesrv
       version: {{ .Chart.AppVersion }}
       app.kubernetes.io/name: {{ include "rocketmq.name" . }}
       app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: rocketmq
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
          - name: TZ
            value: Asia/Shanghai
        command: ["sh","-c",{{ printf "/home/rocketmq/rocketmq-%s/bin/mqnamesrv" $rocketmqVersion }}]

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "rocketmq.fullname" . }}-broker-a
  namespace: {{ .Release.Namespace }}
  labels:
    app: broker
    app.kubernetes.io/component: broker
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  serviceName: {{ include "rocketmq.fullname" . }}-broker-a-hl
  replicas: 1
  selector:
    matchLabels:
      app: broker
      app.kubernetes.io/component: broker
      app.kubernetes.io/name: {{ include "rocketmq.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
     labels:
       app: broker
       app.kubernetes.io/component: broker
       version: {{ .Chart.AppVersion }}
       app.kubernetes.io/name: {{ include "rocketmq.name" . }}
       app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: rocketmq
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
          - name: TZ
            value: Asia/Shanghai
        command: ["sh","-c", {{ printf "/home/rocketmq/rocketmq-%s/bin/mqbroker -c /home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-a.properties" $rocketmqVersion $rocketmqVersion }}]
        volumeMounts:
          - mountPath: /root/logs
            name: rocketmq-data
            subPath: logs
          - mountPath: /root/store
            name: rocketmq-data
            subPath: store
          - name: rocketmq-config
            mountPath: {{ printf "/home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-a.properties" $rocketmqVersion }}
            subPath: broker-a.properties
      volumes:
      - name: rocketmq-config
        configMap:
          name: rocketmq-config
          items:
          - key: broker-a.properties
            path: broker-a.properties
  volumeClaimTemplates:
  - metadata:
      name: rocketmq-data
      namespace: {{ .Release.Namespace }}
      labels:
        app: broker
        app.kubernetes.io/component: broker
        app.kubernetes.io/name: {{ include "rocketmq.name" . }}
        helm.sh/chart: {{ include "rocketmq.chart" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        app.kubernetes.io/managed-by: {{ .Release.Service }}
    spec:
      accessModes:
        - {{ .Values.persistence.accessMode | quote }}
      resources:
        requests:
          storage: {{ .Values.persistence.size | quote }}
      storageClassName: {{ .Values.persistence.storageClass | quote }}

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "rocketmq.fullname" . }}-broker-a-s
  namespace: {{ .Release.Namespace }}
  labels:
    app: broker
    app.kubernetes.io/component: broker
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  serviceName: {{ include "rocketmq.fullname" . }}-broker-a-s-hl
  replicas: 1
  selector:
    matchLabels:
      app: broker
      app.kubernetes.io/component: broker
      app.kubernetes.io/name: {{ include "rocketmq.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
     labels:
       app: broker
       app.kubernetes.io/component: broker
       version: {{ .Chart.AppVersion }}
       app.kubernetes.io/name: {{ include "rocketmq.name" . }}
       app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: rocketmq
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
          - name: TZ
            value: Asia/Shanghai
        command: ["sh","-c", {{ printf "/home/rocketmq/rocketmq-%s/bin/mqbroker -c /home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-a-s.properties" $rocketmqVersion $rocketmqVersion }}]
        volumeMounts:
          - mountPath: /root/logs
            name: rocketmq-data
            subPath: logs
          - mountPath: /root/store
            name: rocketmq-data
            subPath: store
          - name: rocketmq-config
            mountPath: {{ printf "/home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-a-s.properties" $rocketmqVersion }}
            subPath: broker-a-s.properties
      volumes:
      - name: rocketmq-config
        configMap:
          name: rocketmq-config
          items:
          - key: broker-a-s.properties
            path: broker-a-s.properties
  volumeClaimTemplates:
  - metadata:
      name: rocketmq-data
      namespace: {{ .Release.Namespace }}
      labels:
        app: broker
        app.kubernetes.io/component: broker
        app.kubernetes.io/name: {{ include "rocketmq.name" . }}
        helm.sh/chart: {{ include "rocketmq.chart" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        app.kubernetes.io/managed-by: {{ .Release.Service }}
    spec:
      accessModes:
        - {{ .Values.persistence.accessMode | quote }}
      resources:
        requests:
          storage: {{ .Values.persistence.size | quote }}
      storageClassName: {{ .Values.persistence.storageClass | quote }}

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "rocketmq.fullname" . }}-broker-b
  namespace: {{ .Release.Namespace }}
  labels:
    app: broker
    app.kubernetes.io/component: broker
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  serviceName: {{ include "rocketmq.fullname" . }}-broker-b-hl
  replicas: 1
  selector:
    matchLabels:
      app: broker
      app.kubernetes.io/component: broker
      app.kubernetes.io/name: {{ include "rocketmq.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
     labels:
       app: broker
       app.kubernetes.io/component: broker
       version: {{ .Chart.AppVersion }}
       app.kubernetes.io/name: {{ include "rocketmq.name" . }}
       app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: rocketmq
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
          - name: TZ
            value: Asia/Shanghai
        command: ["sh","-c", {{ printf "/home/rocketmq/rocketmq-%s/bin/mqbroker -c /home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-b.properties" $rocketmqVersion $rocketmqVersion }}]
        volumeMounts:
          - mountPath: /root/logs
            name: rocketmq-data
            subPath: logs
          - mountPath: /root/store
            name: rocketmq-data
            subPath: store
          - name: rocketmq-config
            mountPath: {{ printf "/home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-b.properties" $rocketmqVersion }}
            subPath: broker-b.properties
      volumes:
      - name: rocketmq-config
        configMap:
          name: rocketmq-config
          items:
          - key: broker-b.properties
            path: broker-b.properties
  volumeClaimTemplates:
  - metadata:
      name: rocketmq-data
      namespace: {{ .Release.Namespace }}
      labels:
        app: broker
        app.kubernetes.io/component: broker
        app.kubernetes.io/name: {{ include "rocketmq.name" . }}
        helm.sh/chart: {{ include "rocketmq.chart" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        app.kubernetes.io/managed-by: {{ .Release.Service }}
    spec:
      accessModes:
        - {{ .Values.persistence.accessMode | quote }}
      resources:
        requests:
          storage: {{ .Values.persistence.size | quote }}
      storageClassName: {{ .Values.persistence.storageClass | quote }}

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "rocketmq.fullname" . }}-broker-b-s
  namespace: {{ .Release.Namespace }}
  labels:
    app: broker
    app.kubernetes.io/component: broker
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  serviceName: {{ include "rocketmq.fullname" . }}-broker-b-s-hl
  replicas: 1
  selector:
    matchLabels:
      app: broker
      app.kubernetes.io/component: broker
      app.kubernetes.io/name: {{ include "rocketmq.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
     labels:
       app: broker
       app.kubernetes.io/component: broker
       version: {{ .Chart.AppVersion }}
       app.kubernetes.io/name: {{ include "rocketmq.name" . }}
       app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: rocketmq
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
          - name: TZ
            value: Asia/Shanghai
        command: ["sh","-c", {{ printf "/home/rocketmq/rocketmq-%s/bin/mqbroker -c /home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-b-s.properties" $rocketmqVersion $rocketmqVersion }}]
        volumeMounts:
          - mountPath: /root/logs
            name: rocketmq-data
            subPath: logs
          - mountPath: /root/store
            name: rocketmq-data
            subPath: store
          - name: rocketmq-config
            mountPath: {{ printf "/home/rocketmq/rocketmq-%s/conf/2m-2s-async/broker-b-s.properties" $rocketmqVersion }}
            subPath: broker-b-s.properties
      volumes:
      - name: rocketmq-config
        configMap:
          name: rocketmq-config
          items:
          - key: broker-b-s.properties
            path: broker-b-s.properties
  volumeClaimTemplates:
  - metadata:
      name: rocketmq-data
      namespace: {{ .Release.Namespace }}
      labels:
        app: broker
        app.kubernetes.io/component: broker
        app.kubernetes.io/name: {{ include "rocketmq.name" . }}
        helm.sh/chart: {{ include "rocketmq.chart" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        app.kubernetes.io/managed-by: {{ .Release.Service }}
    spec:
      accessModes:
        - {{ .Values.persistence.accessMode | quote }}
      resources:
        requests:
          storage: {{ .Values.persistence.size | quote }}
      storageClassName: {{ .Values.persistence.storageClass | quote }}
```

### 部署 dashboard 容器实例

```bash
# vim /usr/local/k8s/rocketmq/rocketmq/templates/deployment.yaml
```

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "rocketmq.fullname" . }}-dashboard
  namespace: {{ .Release.Namespace }}
  labels:
    app: dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dashboard
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/name: {{ include "rocketmq.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
     labels:
       app: dashboard
       app.kubernetes.io/component: dashboard
       version: {{ .Chart.AppVersion }}
       app.kubernetes.io/name: {{ include "rocketmq.name" . }}
       app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: dashboard
        image: "{{ .Values.dashboard.image.repository }}:{{ .Values.dashboard.image.tag }}"
        imagePullPolicy: {{ .Values.dashboard.image.pullPolicy }}
        env:
          - name: TZ
            value: Asia/Shanghai
          - name: JAVA_OPTS
            value: {{ printf " -Drocketmq.namesrv.addr=%s-namesrv-hl:9876" (include "rocketmq.fullname" .)  }}
```

### 部署容器实例的服务

```bash
# vim /usr/local/k8s/rocketmq/rocketmq/templates/service.yaml 
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "rocketmq.fullname" . }}-namesrv-hl
  labels:
    app: namesrv
    app.kubernetes.io/component: namesrv
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  type: ClusterIP
  sessionAffinity: None
  clusterIP: None
  publishNotReadyAddresses: true
  ports:
    - port: 9876
      targetPort: 9876
      protocol: TCP
      name: rocketmq
  selector:
    app: namesrv
    app.kubernetes.io/component: namesrv
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}

---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "rocketmq.fullname" . }}-dashboard-hl
  labels:
    app: dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    helm.sh/chart: {{ include "rocketmq.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  type: ClusterIP
  sessionAffinity: None
  clusterIP: None
  publishNotReadyAddresses: true
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: dashboard
  selector:
    app: dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}

---
apiVersion: v1
kind: Service
metadata:
  name: rocketmq-cluster-dashboard-service
  namespace: iot
spec:
  selector:
    app: dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/name: {{ include "rocketmq.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  ports:
    - name: dashboard
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/rocketmq/rocketmq-4.9.4.tgz

# helm package /usr/local/k8s/rocketmq/rocketmq -d /usr/local/k8s/rocketmq

# tree /usr/local/k8s/rocketmq
/usr/local/k8s/rocketmq
├── rocketmq
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   ├── serviceaccount.yaml
│   │   ├── service.yaml
│   │   └── statefulset.yaml
│   └── values.yaml
└── rocketmq-4.9.4.tgz
```

## 部署

```bash
# helm install rocketmq-cluster /usr/local/k8s/rocketmq/rocketmq-4.9.4.tgz -n iot
NAME: rocketmq-cluster
LAST DEPLOYED: Thu Aug 18 13:07:29 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace iot -l "app.kubernetes.io/name=rocketmq,app.kubernetes.io/instance=rocketmq-cluster" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace iot $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace iot port-forward $POD_NAME 8080:$CONTAINER_PORT
```

查看资源:

```bash
# kubectl get sts -n iot | grep rocketmq
rocketmq-cluster-broker-a            1/1     8s
rocketmq-cluster-broker-a-s          1/1     8s
rocketmq-cluster-broker-b            1/1     8s
rocketmq-cluster-broker-b-s          1/1     8s
rocketmq-cluster-namesrv             2/2     8s

# kubectl get pod -n iot -o wide | grep rocketmq
rocketmq-cluster-broker-a-0                        1/1     Running     0                30s     10.244.1.49    centos-docker-164   <none>           <none>
rocketmq-cluster-broker-a-s-0                      1/1     Running     0                30s     10.244.2.20    centos-docker-165   <none>           <none>
rocketmq-cluster-broker-b-0                        1/1     Running     0                30s     10.244.2.19    centos-docker-165   <none>           <none>
rocketmq-cluster-broker-b-s-0                      1/1     Running     0                30s     10.244.1.50    centos-docker-164   <none>           <none>
rocketmq-cluster-dashboard-5b85bb5db4-29q8q        1/1     Running     0                30s     10.244.1.47    centos-docker-164   <none>           <none>
rocketmq-cluster-namesrv-0                         1/1     Running     0                30s     10.244.1.48    centos-docker-164   <none>           <none>
rocketmq-cluster-namesrv-1                         1/1     Running     0                28s     10.244.2.18    centos-docker-165   <none>           <none>

# kubectl get pvc,pv -n iot | grep rocketmq
persistentvolumeclaim/rocketmq-data-rocketmq-cluster-broker-a-0     Bound    pvc-362cf32c-25c4-439a-9552-9121199cf81c   10Gi       RWX            nfs-client     49s
persistentvolumeclaim/rocketmq-data-rocketmq-cluster-broker-a-s-0   Bound    pvc-c1df1ed5-0ab9-42ac-bb9b-344d3f2dd046   10Gi       RWX            nfs-client     49s
persistentvolumeclaim/rocketmq-data-rocketmq-cluster-broker-b-0     Bound    pvc-05524f27-84e6-47f9-9e35-db76b340bd2e   10Gi       RWX            nfs-client     49s
persistentvolumeclaim/rocketmq-data-rocketmq-cluster-broker-b-s-0   Bound    pvc-4b88ded1-4f70-4b18-a26c-f5453a4bafa0   10Gi       RWX            nfs-client     49s
persistentvolume/pvc-05524f27-84e6-47f9-9e35-db76b340bd2e   10Gi       RWX            Delete           Bound    iot/rocketmq-data-rocketmq-cluster-broker-b-0     nfs-client              49s
persistentvolume/pvc-362cf32c-25c4-439a-9552-9121199cf81c   10Gi       RWX            Delete           Bound    iot/rocketmq-data-rocketmq-cluster-broker-a-0     nfs-client              49s
persistentvolume/pvc-4b88ded1-4f70-4b18-a26c-f5453a4bafa0   10Gi       RWX            Delete           Bound    iot/rocketmq-data-rocketmq-cluster-broker-b-s-0   nfs-client              49s
persistentvolume/pvc-c1df1ed5-0ab9-42ac-bb9b-344d3f2dd046   10Gi       RWX            Delete           Bound    iot/rocketmq-data-rocketmq-cluster-broker-a-s-0   nfs-client              49s

# kubectl get svc -n iot | grep rocketmq
rocketmq-cluster-dashboard-hl           ClusterIP   None             <none>        8080/TCP                                                                                     69s
rocketmq-cluster-dashboard-service      NodePort    10.110.142.44    <none>        8080:30080/TCP                                                                               69s
rocketmq-cluster-namesrv-hl             ClusterIP   None             <none>        9876/TCP                                                                                     69s
```

## 内部访问 rocketmq 集群

略

## 外部访问 rocketmq 集群

在浏览器的地址栏里输入:```http://192.168.5.163:30080/```
