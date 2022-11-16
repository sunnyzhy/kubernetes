# containerd 拉取 harbor 私有仓库的镜像

***在 kubernetes 所有的节点下执行以下操作。***

##  配置 ssl 证书

```bash
mkdir -p /etc/containerd/core.harbor.domain

cp /xxx/ca.crt /etc/containerd/core.harbor.domain

cp /xxx/tls.crt /etc/containerd/core.harbor.domain

cp /xxx/tls.key /etc/containerd/core.harbor.domain
```

## 修改 containerd 的配置文件

```bash
vim /etc/containerd/config.toml
```

```toml
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."core.harbor.domain".tls]
          insecure_skip_verify = false # 是否跳过证书认证
          ca_file = "/etc/containerd/core.harbor.domain/ca.crt" # CA 证书
          cert_file = "/etc/containerd/core.harbor.domain/tls.crt" # harbor 证书
          key_file = "/etc/containerd/core.harbor.domain/tls.key" # harbor 私钥 
        [plugins."io.containerd.grpc.v1.cri".registry.configs."core.harbor.domain".auth]
          username = "admin"  # 私有仓库的用户名
          password = "adminpassword" # 私有仓库的密码

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."core.harbor.domain"] # 配置私有仓库
          endpoint = ["https://core.harbor.domain/"]
```

harbor 的认证配置有两种方式:

1. 在 ```config.toml``` 里添加配置项 ```[plugins."io.containerd.grpc.v1.cri".registry.configs."core.harbor.domain".auth]```
2. 创建 ```secrect```， 在 ```secrect``` 里配置账号密码

## 重启 containerd 服务

```bash
systemctl daemon-reload

systemctl restart containerd
```

## 查看镜像

```bash
ctr image ls
```

## 拉取镜像

```bash
ctr image pull core.harbor.domain/<NAMESPACE>/<IMAGE_NAME>:<TAG>
```

## 删除镜像

```bash
ctr image rm core.harbor.domain/<NAMESPACE>/<IMAGE_NAME>:<TAG>
```

## 示例

### 使用 ```config.toml``` 里配置的认证信息

创建 ```<STATEFULSET_NAME>.yml```:

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: <STATEFULSET_NAME>
  namespace: <NAMESPACE>
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: <STATEFULSET_NAME>
    spec:
      containers:
        - name: <CONTAINER_NAME>
          image: core.harbor.domain/<NAMESPACE>/<IMAGE>:<TAG>
          imagePullPolicy: IfNotPresent
```

运行 StatefulSet:

```bash
kubectl apply -f <STATEFULSET_NAME>.yml
```

### 创建 ```secrect```

不需要在 ```config.toml``` 里配置认证信息，只需创建 ```secrect```， 并在 ```secrect``` 里配置账号密码。

创建 secrect:

```bash
kubectl create secret docker-registry harbor-pull-secret \
    --docker-server=core.harbor.domain \
    --docker-username=admin \
    --docker-password=adminpassword \
    --namespace=<NAMESPACE>
```

创建 ```<STATEFULSET_NAME>.yml```:

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: <STATEFULSET_NAME>
  namespace: <NAMESPACE>
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: <STATEFULSET_NAME>
    spec:
      imagePullSecrets:
        - name: harbor-pull-secret
      containers:
        - name: <CONTAINER_NAME>
          image: core.harbor.domain/<NAMESPACE>/<IMAGE>:<TAG>
          imagePullPolicy: IfNotPresent
```

***注: 在 imagePullSecrets 里指明 secrect 的 name***

运行 StatefulSet:

```bash
kubectl apply -f <STATEFULSET_NAME>.yml
```
