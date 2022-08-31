# helm - ingress-nginx 集群

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

## 创建 ingress-nginx 目录

```bash
# mkdir -p /usr/local/k8s/ingress-nginx
```

## 查找 chart

```bash
# helm search repo ingress-nginx
NAME               	CHART VERSION	APP VERSION	DESCRIPTION                                       
nginx/ingress-nginx	4.2.3        	1.3.0      	Ingress controller for Kubernetes using NGINX a...
```

## 拉取 chart

```bash
# helm pull nginx/ingress-nginx -d /usr/local/k8s/ingress-nginx
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/ingress-nginx/ingress-nginx-4.2.3.tgz -C /usr/local/k8s/ingress-nginx
```

### 修改 ingress-nginx 配置

```bash
# vim /usr/local/k8s/ingress-nginx/ingress-nginx/values.yaml
```

```bash
controller:
  hostPort:
    enabled: true

  service:
    type: ClusterIP
```

```bash
# sed -i 's+registry: registry.k8s.io+registry: registry.aliyuncs.com+' /usr/local/k8s/ingress-nginx/ingress-nginx/values.yaml

# sed -i 's+image: ingress-nginx\/controller+image: google_containers\/nginx-ingress-controller+' /usr/local/k8s/ingress-nginx/ingress-nginx/values.yaml

# sed -i 's+image: ingress-nginx\/kube-webhook-certgen+image: google_containers\/kube-webhook-certgen+' /usr/local/k8s/ingress-nginx/ingress-nginx/values.yaml
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/ingress-nginx/ingress-nginx-4.2.3.tgz

# helm package  /usr/local/k8s/ingress-nginx/ingress-nginx -d  /usr/local/k8s/ingress-nginx
```

## 部署

```bash
# helm install ingress-nginx-cluster /usr/local/k8s/ingress-nginx/ingress-nginx-4.2.3.tgz
NAME: ingress-nginx-cluster
LAST DEPLOYED: Wed Aug 31 16:45:27 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
Get the application URL by running these commands:
  export POD_NAME=$(kubectl --namespace default get pods -o jsonpath="{.items[0].metadata.name}" -l "app=ingress-nginx,component=controller,release=ingress-nginx-cluster")
  kubectl --namespace default port-forward $POD_NAME 8080:80
  echo "Visit http://127.0.0.1:8080 to access your application."

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

查看资源:

```bash
# kubectl get deploy | grep ingress-nginx
ingress-nginx-cluster-controller   2/2     2            2           108s

# kubectl get rs | grep ingress-nginx
ingress-nginx-cluster-controller-66bf6fb7f6   2         2         2       2m17s

# kubectl get pod -o wide | grep ingress-nginx
ingress-nginx-cluster-controller-66bf6fb7f6-cx9m2   1/1     Running   0          2m30s   10.244.1.142   centos-docker-164   <none>           <none>
ingress-nginx-cluster-controller-66bf6fb7f6-zhztm   1/1     Running   0          2m30s   10.244.2.131   centos-docker-165   <none>           <none>

# kubectl get svc | grep ingress-nginx
ingress-nginx-cluster-controller             ClusterIP   10.108.17.37     <none>        80/TCP,443/TCP   3m5s
ingress-nginx-cluster-controller-admission   ClusterIP   10.111.191.206   <none>        443/TCP          3m5s
```

## 内部访问 ingress-nginx 集群

略

## 外部访问 ingress-nginx 集群

略

## FAQ

### Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: IngressClass "nginx" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata;

原因: 已存在名称为 ```nginx``` 的资源

```bash
# kubectl get IngressClass --all-namespaces
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       10m
```

解决方法: 删除已经存在的 ```nginx``` 的资源，再 install
