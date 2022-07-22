# ingress 示例

## 前言

[安装 ingress-nginx](./安装ingress-nginx.md '安装ingress-nginx')

查询 ```ingress-nginx-controller```:

```bash
# kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-qwt4b        0/1     Completed   0          18h
ingress-nginx-admission-patch-ffh4p         0/1     Completed   0          18h
ingress-nginx-controller-68cccc75c6-smwmw   1/1     Running     0          18h

# kubectl describe pod -n ingress-nginx ingress-nginx-controller-68cccc75c6-smwmw | grep -E 'Ports:|Host Ports:'
    Ports:         80/TCP, 443/TCP, 8443/TCP
    Host Ports:    80/TCP, 443/TCP, 8443/TCP
```

- Ports: 内部端口，只能在集群内部访问
- Host Ports: 主机端口，供集群外部访问

## 部署 Deployment 和 Pod

### 创建 deploy.yaml

```bash
# mkdir -p /usr/local/k8s/ingress

# vim /usr/local/k8s/ingress/deploy.yaml
```

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: ns-nginx

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
  namespace: ns-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 部署 Deployment 和 Pod

```bash
# kubectl apply -f /usr/local/k8s/ingress/deploy.yaml

# kubectl get deployments -n ns-nginx
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           57s

# kubectl get pods -n ns-nginx -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
nginx-deployment-544dc8b7c4-4bxkq   1/1     Running   0          4m51s   10.244.2.18   centos-docker-165   <none>           <none>
nginx-deployment-544dc8b7c4-pzjf5   1/1     Running   0          4m51s   10.244.1.26   centos-docker-164   <none>           <none>
nginx-deployment-544dc8b7c4-sxc8g   1/1     Running   0          4m51s   10.244.1.27   centos-docker-164   <none>           <none>
```

## 部署 Service

### 创建 service.yaml

```bash
# mkdir -p /usr/local/k8s/ingress

# vim /usr/local/k8s/ingress/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: ns-nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 部署 Service

```bash
# kubectl apply -f /usr/local/k8s/ingress/service.yaml

# kubectl get services -n ns-nginx
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.111.200.189   <none>        80/TCP    9s
```

## 部署 Ingress

### 创建 ingress.yaml

```bash
# mkdir -p /usr/local/k8s/ingress

# vim /usr/local/k8s/ingress/ingress.yaml
```

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
  namespace: ns-nginx
spec:
  rules:
  - host: k8sdemo.ingress.com
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### 部署 Ingress

```bash
# kubectl apply -f /usr/local/k8s/ingress/ingress.yaml

# kubectl get ingress -n ns-nginx
NAME            CLASS    HOSTS                 ADDRESS   PORTS   AGE
nginx-ingress   <none>   k8sdemo.ingress.com             80      22s

# kubectl describe ingress nginx-ingress -n ns-nginx
Name:             nginx-ingress
Labels:           <none>
Namespace:        ns-nginx
Address:          10.109.253.191
Ingress Class:    <none>
Default backend:  <default>
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  k8sdemo.ingress.com  
                       /   nginx-service:80 (10.244.1.26:80,10.244.1.27:80,10.244.2.18:80)
Annotations:           kubernetes.io/ingress.class: nginx
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    15s (x2 over 64s)  nginx-ingress-controller  Scheduled for sync
```

## HTTP 访问

### 修改 hosts

***在任一主机的 hosts 文件里配置域名映射: ```<ingress-nginx-controller 所在节点的 IP 地址>   k8sdemo.ingress.com```***

查询 ```ingress-nginx-controller``` 所在的 Node 和 IP:

```bash
# kubectl describe pod -n ingress-nginx ingress-nginx-controller-68cccc75c6-smwmw | grep -E 'Node:|IP:'
Node:         centos-docker-165/192.168.5.165
IP:           192.168.5.165
  IP:           192.168.5.165
```

在任一主机的 hosts 文件里配置域名映射:

```bash
# cat << EOF >> /etc/hosts
> 192.168.5.165   k8sdemo.ingress.com
> EOF
```

### HTTP 访问

在配置了 hosts 域名映射的在任一主机上访问:

```bash
# curl k8sdemo.ingress.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
