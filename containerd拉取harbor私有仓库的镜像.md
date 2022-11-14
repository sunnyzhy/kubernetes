# containerd 拉取 harbor 私有仓库的镜像

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

## 重启 containerd 服务

```bash
systemctl daemon-reload

systemctl restart containerd
```
