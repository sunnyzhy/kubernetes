# kubernetes 里不同的 port 的区别

## 区别

- containerPort: 是 pod 内部容器的端口，在 Pod 和 Deployment 类型的模板里

- port: 是 kubernetes 集群内部访问 service 的端口，通过 ```clusterIP:port``` 可以访问到某个 service，在 Service 类型的模板里

- nodePort: 是外部访问 kubernetes 集群中 service 的端口，通过 ```nodeIP:nodePort``` 可以从外部访问到某个 service，在 Service 类型的模板里

- targetPort: 是 kubernetes 集群内部 service 访问 pod 的端口，通过 ```podIP:targetPort``` 可以访问到某个 pod，在 Service 类型的模板里
   ***注: targetPort 映射的端口必须是 containerPort 端口***

## 示例

deployment.yaml 配置的内容如下:

```yml
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

应用到 kubernetes 之后，pod 的 IP 列表为:

|IP|
|--|
|10.244.1.11|
|10.244.2.11|
|10.244.2.10|

访问 ```10.244.1.11:80```:

```bash
# curl 10.244.1.11:80
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

service.yaml 配置的内容如下:

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
      targetPort: 9376
```

应用到 kubernetes 之后，service 的 IP 与 Endpoints 为:

|Name|Cluster IP|Endpoints|
|--|--|--|
|nginx-service|10.111.25.14|10.244.1.11:9376, 10.244.2.10:9376, 10.244.2.11:9376|

访问 ```10.111.25.14```:

```bash
# curl 10.111.25.14:80
curl: (7) Failed connect to 10.111.25.14:80; 拒绝连接
```

此时查看 IPVS:

```bash
# ipvsadm -ln
TCP  10.111.25.14:80 rr
  -> 10.244.1.11:9376             Masq    1      0          0         
  -> 10.244.2.10:9376             Masq    1      0          0         
  -> 10.244.2.11:9376             Masq    1      0          0        
```

```Cluster IP:80``` 映射的是 ```Pod IP:9376```，而 Pod 并没有开启 9376 端口，所以访问拒绝。

修改 service.yaml 配置的 ```targetPort``` 为 ```80```:

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

再次访问 ```10.111.25.14``` 成功:

```bash
# curl 10.111.25.14:80
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

此时查看 IPVS:

```bash
# ipvsadm -ln
TCP  10.111.25.14:80 rr
  -> 10.244.1.11:80               Masq    1      0          0         
  -> 10.244.2.10:80               Masq    1      0          0         
  -> 10.244.2.11:80               Masq    1      0          0          
```

## 注意事项

1. pod、deployment、service 必须位于同一命名空间下
2. service 的 selector 必须是在 pod 和 deployment 里定义的标签的 ```key: value```
3. service 的 targetPort 映射的端口必须是 pod 和 deployment 的 containerPort 端口
