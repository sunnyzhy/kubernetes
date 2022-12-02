# headless 服务示例

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-headless # headless 服务的名称
  namespace: ns # 命名空间
spec:
  selector:
    app: nginx # 服务关联的 Pod 标签
  type: ClusterIP # 服务类型为 ClusterIP
  clusterIP: None # 定义为 headless 服务
  ports:
    - protocol: TCP
      port: 80 # 服务在集群内部的端口
      targetPort: 80 # 服务关联的 Pod 端口

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-cluster # 控制器的名称
  namespace: ns # 命名空间
spec:
  replicas: 2
  serviceName: nginx-service-headless # 控制器关联的 headless 服务
  selector:
    matchLabels:
      app: nginx # 控制器关联的 Pod 标签
  template:
    metadata:
      labels:
        app: nginx # Pod 标签
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80 # Pod 端口
```
