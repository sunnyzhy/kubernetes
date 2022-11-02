# kubernetes 命令

## kubectl

### 列出所有命名空间

```bash
kubectl get namespaces
```

### 列出所有 service

```bash
kubectl get services --all-namespaces
```

### 列出指定命名空间下的 service

```bash
kubectl get services -o wide -n <namespace-name>
```

### 列出所有 deployment

```bash
kubectl get deployments --all-namespaces
```

### 列出指定命名空间下的 deployment

```bash
kubectl get deployments -o wide -n <namespace-name>
```

### 列出所有 pod

```bash
kubectl get pods --all-namespaces
```

### 列出指定命名空间下的 pod

```bash
kubectl get pods -o wide -n <namespace-name>
```

### 查看 pod 详情

```bash
kubectl describe pod <pod-name> -n <namespace-name>
```

### 进入 pod

```bash
kubectl exec -it -n <namespace-name> <pod-name> -- /bin/sh
```

### 查看 pod 日志

```bash
kubectl logs -n <namespace-name> <pod-name>
```
