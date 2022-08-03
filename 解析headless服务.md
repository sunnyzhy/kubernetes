# 解析 headless 服务

## 查找 kube-dns 在集群内部的 CLUSTER-IP

```bash
kubectl get svc -n kube-system
```

```bash
kubectl get svc -n kube-system | grep 'kube-dns' | awk '{print $3}'
```

## 解析 headless 服务

### 通过 ```nslookup``` 命令解析

```bash
nslookup <headless_service_name>.<namespace>.svc.cluster.local <kube-system CLUSTER-IP>
```

### 通过 ```dig``` 命令解析

```bash
dig <headless_service_name>.<namespace>.svc.cluster.local @<kube-system CLUSTER-IP>
```

```bash
dig <headless_service_name>.<namespace>.svc.cluster.local @<kube-system CLUSTER-IP> +nocomments +noquestion +noauthority +noadditional +nostats
```

```bash
dig <headless_service_name>.<namespace>.svc.cluster.local @<kube-system CLUSTER-IP> +short
```
