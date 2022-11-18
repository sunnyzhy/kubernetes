# containerd - 拉取 harbor 仓库里的镜像

***注: 在 kubernetes 所有的节点下执行以下操作。***

##  配置 ssl 证书

```bash
mkdir -p /etc/containerd/certs.d/core.harbor.domain

cp /xxx/harbor/ca.crt /etc/containerd/certs.d/core.harbor.domain

cp /xxx/harbor/tls.crt /etc/containerd/certs.d/core.harbor.domain

cp /xxx/harbor/tls.key /etc/containerd/certs.d/core.harbor.domain
```

## 修改 containerd 的配置文件

```bash
vim /etc/containerd/config.toml
```

```toml
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."core.harbor.domain".tls]
          insecure_skip_verify = false # 是否跳过证书认证
          ca_file = "/etc/containerd/certs.d/core.harbor.domain/ca.crt" # CA 证书
          cert_file = "/etc/containerd/certs.d/core.harbor.domain/tls.crt" # harbor 证书
          key_file = "/etc/containerd/certs.d/core.harbor.domain/tls.key" # harbor 私钥 
        [plugins."io.containerd.grpc.v1.cri".registry.configs."core.harbor.domain".auth]
          username = "admin"  # 私有仓库的用户名
          password = "adminpassword" # 私有仓库的密码

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."core.harbor.domain"] # 配置私有仓库
          endpoint = ["https://core.harbor.domain/v2"]
```

***注：如果 ```harbor``` 使用的 ```API``` 是 ```Harbor API V2.0```，就需要在镜像的 ```endpoint``` 末尾加 ```/v2```***

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
ctr -n k8s.io image ls
```

## 推送镜像

```bash
ctr -n k8s.io tag <IMAGE_NAME>:<TAG> core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>

ctr -n k8s.io image push --user <USERNAME>:<PASSWORD> -k core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>

ctr -n k8s.io image push --user <USERNAME>:<PASSWORD> --tlscacert ca.crt core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>
```

## 拉取镜像

在 ```config.toml``` 里配置了 ```tls```:

```bash
ctr -n k8s.io image pull core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>
```

没有在 ```config.toml``` 里配置 ```tls```:

```bash
ctr -n k8s.io image pull --user <USERNAME>:<PASSWORD> -k core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>

ctr -n k8s.io image pull --user <USERNAME>:<PASSWORD> --tlscacert ca.crt core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>
```

## 删除镜像

```bash
ctr -n k8s.io image rm core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>
```

## 查看日志

```bash
journalctl -u containerd
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
          image: core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE>:<TAG>
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
          image: core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE>:<TAG>
          imagePullPolicy: IfNotPresent
```

***注: 在 imagePullSecrets 里指明 secrect 的 name***

运行 StatefulSet:

```bash
kubectl apply -f <STATEFULSET_NAME>.yml
```

***如果拉取镜像失败，请参考 [crictl-拉取harbor仓库里的镜像#FAQ](https://github.com/sunnyzhy/kubernetes/blob/main/crictl-%E6%8B%89%E5%8F%96harbor%E4%BB%93%E5%BA%93%E9%87%8C%E7%9A%84%E9%95%9C%E5%83%8F.md#faq 'crictl-拉取harbor仓库里的镜像#FAQ')***
