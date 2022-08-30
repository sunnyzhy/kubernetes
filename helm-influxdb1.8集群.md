# helm - influxdb1.8 集群

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
    https://github.com/influxtsdb/helm-charts
    
    https://github.com/chengshiwen/influxdb-cluster
    ```

## 创建 influxdb 目录

```bash
# mkdir -p /usr/local/k8s/influxdb/influxdb-cluster
```

## 拉取 chart

```bash
# git clone https://github.com/influxtsdb/helm-charts.git /usr/local/k8s/influxdb/influxdb-cluster
```

## 修改 chart 配置

### 修改 influxdb 配置

```bash
# vim /usr/local/k8s/influxdb/influxdb-cluster/charts/influxdb-cluster/values.yaml
```

```yml
meta:
  persistence:
    storageClass: "nfs-client"

data:
  service:
    nodePorts:
      nodePort: 30086
  ingress:
    enabled: true
    className: "nginx"
    hosts:
      - host: iot.influx
  persistence:
    storageClass: "nfs-client"
```

#### 部署 service 实例

在 ```service.yaml``` 里添加 NodePort 类型的 Service，对外提供服务（非必需，功能同 ingress）:

```bash
# vim /usr/local/k8s/influxdb/influxdb-cluster/charts/influxdb-cluster/templates/data-service.yaml
```

```yml

---
apiVersion: v1
kind: Service
metadata:
{{- if .Values.data.service.annotations }}
  annotations:
{{ toYaml .Values.data.service.annotations | indent 4 }}
{{- end }}
  name: {{ template "influxdb-cluster.fullname" . }}-data-hl
  labels:
    influxdb.cluster/component: data
    {{- include "influxdb-cluster.labels" . | nindent 4 }}
spec:
  type: NodePort
  ports:
  - port: 8086
    protocol: TCP
    name: http
    nodePort: {{ .Values.data.service.nodePorts.nodePort }}
  selector:
    influxdb.cluster/component: data
{{- include "influxdb-cluster.selectorLabels" . | nindent 4 }}
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/influxdb/influxdb-cluster-0.1.0.tgz

# helm package /usr/local/k8s/influxdb/influxdb-cluster/charts/influxdb-cluster -d /usr/local/k8s/influxdb
```

## 部署

```bash
# helm install influxdb-cluster /usr/local/k8s/influxdb/influxdb-cluster-0.1.0.tgz -n iot
NAME: influxdb-cluster
LAST DEPLOYED: Tue Aug 30 09:52:41 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the InfluxDB Cluster HTTP URL by running these commands:
  http://iot.influx/
By default, InfluxDB Cluster can be accessed via port 8086 on the following DNS name from within your kubernetes cluster:
  http://influxdb-cluster-data.iot:8086
```

查看资源:

```bash
# kubectl get statefulset -n iot | grep influxdb-cluster
influxdb-cluster-data                2/2     96s
influxdb-cluster-meta                3/3     96s

# kubectl get pod -n iot -o wide | grep influxdb-cluster
influxdb-cluster-data-0                            1/1     Running   0               2m1s    10.244.1.111   centos-docker-164   <none>           <none>
influxdb-cluster-data-1                            1/1     Running   0               2m1s    10.244.2.101   centos-docker-165   <none>           <none>
influxdb-cluster-meta-0                            1/1     Running   0               2m1s    10.244.1.112   centos-docker-164   <none>           <none>
influxdb-cluster-meta-1                            1/1     Running   0               2m1s    10.244.2.100   centos-docker-165   <none>           <none>
influxdb-cluster-meta-2                            1/1     Running   0               2m1s    10.244.1.113   centos-docker-164   <none>           <none>

# kubectl get pvc,pv -n iot | grep influxdb-cluster
persistentvolumeclaim/influxdb-cluster-data-data-influxdb-cluster-data-0   Bound    pvc-e4d754aa-3f1e-48f1-be5a-b2220a6b53d0   25Gi       RWO            nfs-client     2m16s
persistentvolumeclaim/influxdb-cluster-data-data-influxdb-cluster-data-1   Bound    pvc-87c81f05-747a-44db-a4e3-164c8f60ab73   25Gi       RWO            nfs-client     2m16s
persistentvolumeclaim/influxdb-cluster-meta-data-influxdb-cluster-meta-0   Bound    pvc-aad54acf-acb7-43e5-8394-fe69da33a442   25Gi       RWO            nfs-client     2m16s
persistentvolumeclaim/influxdb-cluster-meta-data-influxdb-cluster-meta-1   Bound    pvc-de167119-4bca-4b2c-8072-f521c475fa39   25Gi       RWO            nfs-client     2m16s
persistentvolumeclaim/influxdb-cluster-meta-data-influxdb-cluster-meta-2   Bound    pvc-45ae054c-3cb1-4d7e-a6c8-0bc34ba1185c   25Gi       RWO            nfs-client     2m16s
persistentvolume/pvc-45ae054c-3cb1-4d7e-a6c8-0bc34ba1185c   25Gi       RWO            Delete           Bound    iot/influxdb-cluster-meta-data-influxdb-cluster-meta-2   nfs-client              2m15s
persistentvolume/pvc-87c81f05-747a-44db-a4e3-164c8f60ab73   25Gi       RWO            Delete           Bound    iot/influxdb-cluster-data-data-influxdb-cluster-data-1   nfs-client              2m16s
persistentvolume/pvc-aad54acf-acb7-43e5-8394-fe69da33a442   25Gi       RWO            Delete           Bound    iot/influxdb-cluster-meta-data-influxdb-cluster-meta-0   nfs-client              2m16s
persistentvolume/pvc-de167119-4bca-4b2c-8072-f521c475fa39   25Gi       RWO            Delete           Bound    iot/influxdb-cluster-meta-data-influxdb-cluster-meta-1   nfs-client              2m16s
persistentvolume/pvc-e4d754aa-3f1e-48f1-be5a-b2220a6b53d0   25Gi       RWO            Delete           Bound    iot/influxdb-cluster-data-data-influxdb-cluster-data-0   nfs-client              2m16s

# kubectl get svc -n iot | grep influxdb-cluster
influxdb-cluster-data                   ClusterIP   None             <none>        8086/TCP,8088/TCP,2003/TCP,4242/TCP,25826/UDP,8089/UDP                        2m40s
influxdb-cluster-data-hl                NodePort    10.108.194.142   <none>        8086:30086/TCP,8088:32678/TCP                                                 2m40s
influxdb-cluster-meta                   ClusterIP   None             <none>        8089/TCP,8091/TCP                                                             2m40s

# kubectl get ingress -n iot | grep influxdb-cluster
influxdb-cluster-data          nginx   iot.influx          10.108.27.226   80      2m59s
```

## 添加集群节点

```bash
# kubectl exec -it influxdb-cluster-meta-0 -n iot -- /bin/sh

# influxd-ctl add-meta influxdb-cluster-meta-0.influxdb-cluster-meta:8091

# influxd-ctl add-meta influxdb-cluster-meta-1.influxdb-cluster-meta:8091

# influxd-ctl add-meta influxdb-cluster-meta-2.influxdb-cluster-meta:8091

# influxd-ctl add-data influxdb-cluster-data-0.influxdb-cluster-data:8088

# influxd-ctl add-data influxdb-cluster-data-1.influxdb-cluster-data:8088

# influxd-ctl show
Data Nodes
==========
ID	TCP Address
4 	 influxdb-cluster-data-0.influxdb-cluster-data:8088
5 	 influxdb-cluster-data-1.influxdb-cluster-data:8088

Meta Nodes
==========
ID	TCP Address
1 	 influxdb-cluster-meta-0.influxdb-cluster-meta:8091
2 	 influxdb-cluster-meta-1.influxdb-cluster-meta:8091
3 	 influxdb-cluster-meta-2.influxdb-cluster-meta:8091
```

## 内部访问 influxdb 集群

```bash
# curl -XPOST "http://10.108.194.142:8086/query" --data-urlencode "q=CREATE DATABASE bar"

# curl -XPOST "http://10.108.194.142:8086/write?db=bar" -d 'cpu,host=server01,region=uswest load=42 1434055562000000000'

# curl -G "http://10.108.194.142:8086/query?pretty=true" --data-urlencode "db=bar" --data-urlencode "q=SELECT * FROM cpu WHERE host='server01' AND time < now() - 1d"
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu",
                    "columns": [
                        "time",
                        "host",
                        "load",
                        "region"
                    ],
                    "values": [
                        [
                            "2015-06-11T20:46:02Z",
                            "server01",
                            42,
                            "uswest"
                        ]
                    ]
                }
            ]
        }
    ]
}

# curl -XPOST "http://10.108.194.142:8086/query" --data-urlencode "q=DROP DATABASE bar"
```

## 外部访问 influxdb 集群

### 通过对外 Service 的方式

```bash
# curl -XPOST "http://192.168.5.163:30086/query" --data-urlencode "q=CREATE DATABASE bar"

# curl -XPOST "http://192.168.5.163:30086/write?db=bar" -d 'cpu,host=server01,region=uswest load=42 1434055562000000000'

# curl -G "http://192.168.5.163:30086/query?pretty=true" --data-urlencode "db=bar" --data-urlencode "q=SELECT * FROM cpu WHERE host='server01' AND time < now() - 1d"
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu",
                    "columns": [
                        "time",
                        "host",
                        "load",
                        "region"
                    ],
                    "values": [
                        [
                            "2015-06-11T20:46:02Z",
                            "server01",
                            42,
                            "uswest"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

### 通过 ingress 的方式

在任一外部服务器的 hosts 文件里配置域名映射:

```bash
# vim /etc/hosts
192.168.5.165 iot.influx
```

```bash
# curl -XPOST "http://iot.influx/query" --data-urlencode "q=CREATE DATABASE foo"

# curl -XPOST "http://iot.influx/write?db=foo" -d 'cpu,host=server02,region=uswest load=42 1434055562000000000'

# curl -G "http://iot.influx/query?pretty=true" --data-urlencode "db=foo" --data-urlencode "q=SELECT * FROM cpu WHERE host='server02' AND time < now() - 1d"
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu",
                    "columns": [
                        "time",
                        "host",
                        "load",
                        "region"
                    ],
                    "values": [
                        [
                            "2015-06-11T20:46:02Z",
                            "server02",
                            42,
                            "uswest"
                        ]
                    ]
                }
            ]
        }
    ]
}
```
