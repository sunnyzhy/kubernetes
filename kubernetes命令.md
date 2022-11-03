# kubernetes 命令

## kubeadm

kubeadm 用于初始化 Cluster 。

### 创建一个 Master 节点

```bash
kubeadm init --control-plane-endpoint=<ip> --pod-network-cidr=10.244.0.0/16 --image-repository registry.aliyuncs.com/google_containers
```

### 将一个 Node 节点加入到集群中

```bash
kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash> --control-plane 
```

## kubectl

kubectl 是 Kubernetes 命令行工具。通过 kubectl 可以部署和管理应用，查看资源，创建、删除和更新组件。

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

## kubelet

kubelet 运行在 Cluster 所有节点上，负责启动 Pod 和容器。
