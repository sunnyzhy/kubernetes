# emqx 集群 - helm

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

## 创建 emqx 目录

```bash
# mkdir -p /usr/local/k8s/emqx
```

## 添加 chart

```bash
# helm repo add emqx https://repos.emqx.io/charts

# helm repo update
```

## 查找 chart

```bash
# helm search repo emqx
NAME              	CHART VERSION	APP VERSION	DESCRIPTION                              
emqx/emqx         	5.0.3        	5.0.3      	A Helm chart for EMQX                    
emqx/emqx-ee      	4.4.6        	4.4.6      	A Helm chart for EMQ X                   
emqx/emqx-operator	1.0.10       	1.2.5      	A Helm chart for EMQX Operator Controller
emqx/kuiper       	0.9.0        	0.9.0      	A lightweight IoT edge analytic software 
```

## 拉取 chart

```bash
# helm pull emqx/emqx -d /usr/local/k8s/emqx
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/emqx/emqx-5.0.3.tgz -C /usr/local/k8s/emqx

# vim /usr/local/k8s/emqx/emqx/values.yaml
```

```yml
emqxConfig:
  EMQX_DASHBOARD__DEFAULT_PASSWORD: root
  EMQX_DASHBOARD__DEFAULT_USERNAME: admin

persistence:
  enabled: true
  storageClassName: "nfs-client"

tolerations: 
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/emqx/emqx-5.0.3.tgz

# helm package /usr/local/k8s/emqx/emqx -d /usr/local/k8s/emqx

# tree /usr/local/k8s/emqx
/usr/local/k8s/emqx
├── emqx
│   ├── Chart.yaml
│   ├── README.md
│   ├── templates
│   │   ├── configmap.yaml
│   │   ├── _helpers.tpl
│   │   ├── ingress.yaml
│   │   ├── rbac.yaml
│   │   ├── secret.yaml
│   │   ├── service-monitor.yaml
│   │   ├── service.yaml
│   │   └── StatefulSet.yaml
│   └── values.yaml
└── emqx-5.0.3.tgz
```

## 部署

```bash
# helm install emqx-cluster /usr/local/k8s/emqx/emqx-5.0.3.tgz -n iot
NAME: emqx-cluster
LAST DEPLOYED: Thu Aug 11 15:43:47 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

查看资源:

```bash
# kubectl get sts -n iot | grep emqx
emqx-cluster                         3/3     115s

# kubectl get pod -n iot -o wide | grep emqx
emqx-cluster-0                                     1/1     Running            0                4m4s   10.244.1.112   centos-docker-164   <none>           <none>
emqx-cluster-1                                     1/1     Running            0                4m4s   10.244.2.89    centos-docker-165   <none>           <none>
emqx-cluster-2                                     1/1     Running            0                4m4s   10.244.1.111   centos-docker-164   <none>           <none>

# kubectl get pvc,pv -n iot | grep emqx
persistentvolumeclaim/emqx-data-emqx-cluster-0                Bound    pvc-96306b7a-a802-4ea7-8b89-afd7c839159d   20Mi       RWO            nfs-client     11m
persistentvolumeclaim/emqx-data-emqx-cluster-1                Bound    pvc-4019a423-b3f9-4740-af77-1eccccb0ad2c   20Mi       RWO            nfs-client     11m
persistentvolumeclaim/emqx-data-emqx-cluster-2                Bound    pvc-fd38d0c9-e7a5-4afe-9609-70afc7dc7e52   20Mi       RWO            nfs-client     11m
persistentvolume/pvc-4019a423-b3f9-4740-af77-1eccccb0ad2c   20Mi       RWO            Delete           Bound    iot/emqx-data-emqx-cluster-1                nfs-client              11m
persistentvolume/pvc-96306b7a-a802-4ea7-8b89-afd7c839159d   20Mi       RWO            Delete           Bound    iot/emqx-data-emqx-cluster-0                nfs-client              11m
persistentvolume/pvc-fd38d0c9-e7a5-4afe-9609-70afc7dc7e52   20Mi       RWO            Delete           Bound    iot/emqx-data-emqx-cluster-2                nfs-client              11m

# kubectl get svc -n iot | grep emqx
emqx-cluster                            ClusterIP   10.99.86.156     <none>        1883/TCP,8883/TCP,8083/TCP,8084/TCP,18083/TCP                                                4m48s
emqx-cluster-headless                   ClusterIP   None             <none>        1883/TCP,8883/TCP,8083/TCP,8084/TCP,18083/TCP,4370/TCP                                       4m48s
```

## 内部访问 emqx 集群

略

## 外部访问 emqx 集群

创建 NodePort 类型的 Service:

```bash
# vim /usr/local/k8s/emqx/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: emqx-cluster-service
  namespace: iot
spec:
  selector:
    app.kubernetes.io/instance: emqx-cluster
    app.kubernetes.io/name: emqx
  ports:
    - name: mqtt
      protocol: TCP
      port: 1883
      targetPort: mqtt
      nodePort: 30183
    - name: mqttssl
      protocol: TCP
      port: 8883
      targetPort: mqttssl
      nodePort: 30883
    - name: ws
      protocol: TCP
      port: 8083
      targetPort: ws
      nodePort: 30083
    - name: wss
      protocol: TCP
      port: 8084
      targetPort: wss
      nodePort: 30084
    - name: dashboard
      protocol: TCP
      port: 18083
      targetPort: dashboard
      nodePort: 31083
    - name: ekka
      protocol: TCP
      port: 4370
      targetPort: ekka
      nodePort: 30370
  type: NodePort
```

```bash
# kubectl apply -f /usr/local/k8s/emqx/service.yaml

# kubectl get svc emqx-cluster-service -n iot
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                      AGE
emqx-cluster-service   NodePort   10.96.246.44   <none>        1883:30183/TCP,8883:30883/TCP,8083:30083/TCP,8084:30084/TCP,18083:31083/TCP,4370:30370/TCP   17s
```

外部服务器访问 emqx 的 dashboard:

在浏览器的地址栏里输入:```http://192.168.5.163:31083/```; ```用户名/密码```: ```admin/root```

普通连接 emqx 集群:

[emqx-01](./images/emqx/emqx-01.png)

ssl 连接 emqx 集群:

1. 从 pod 里拷贝 certs 目录:

    ```bash
    # kubectl exec -n iot emqx-cluster-0 -- tar cf - /opt/emqx/etc/certs | tar xf - -C /usr/local/k8s/emqx/
    ```

2. 连接 emqx 集群:

    [emqx-02](./images/emqx/emqx-02.png)
