# crictl - 拉取 harbor 仓库里的镜像

***注: 在 kubernetes 所有的节点下执行以下操作。***

##  配置 ```crictl.yaml```

```bash
# crictl config runtime-endpoint unix:///run/containerd/containerd.sock

# crictl config image-endpoint unix:///run/containerd/containerd.sock

# systemctl daemon-reload

# cat /etc/crictl.yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
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
crictl image ls
```

## 推送镜像

```crictl``` 没有推送镜像的功能。

## 拉取镜像

```bash
crictl pull core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>

crictl pull --creds <USERNAME>:<PASSWORD> core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>
```

## 删除镜像

```bash
crictl rmi core.harbor.domain/<HARBOR_PROJECT_NAME>/<IMAGE_NAME>:<TAG>
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

## FAQ

### 问题 1

执行 ```crictl pull``` 时出现以下警告信息:

```
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
```

原因: 由于 ```crictl``` 不知道使用哪个 ```sock``` 导致的

解决方法:

```bash
# crictl config runtime-endpoint unix:///run/containerd/containerd.sock

# crictl config image-endpoint unix:///run/containerd/containerd.sock

# systemctl daemon-reload

# cat /etc/crictl.yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
```

### 问题 2

执行 ```crictl pull``` 时出现以下错误信息:

```
FATA[0000] pulling image: rpc error: code = NotFound desc = failed to pull and unpack image "core.harbor.domain/iot/busybox:latest": failed to unpack image on snapshotter overlayfs: unexpected media type text/html for sha256:515882a4af328b4195c94fb9398ac2325fa28674d7587084b3d5d633a1cefe59: not found
```

原因: ```harbor``` 使用的 ```API``` 是 ```Harbor API V2.0```

解决方法:

```bash
vim /etc/containerd/config.toml
```

```toml
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."core.harbor.domain"]
          endpoint = ["https://core.harbor.domain/v2"]
```

***注：在 ```endpoint``` 的末尾加 ```/v2```***

参考： [github issues](https://github.com/k3s-io/k3s/issues/5502 'github issues')
