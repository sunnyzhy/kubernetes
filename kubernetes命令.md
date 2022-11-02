# kubernetes 命令

## kubeadm

kubeadm 是官方社区推出的一个用于快速部署 kubernetes 集群的工具。

这个工具能通过两条指令完成一个 kubernetes 集群的部署: ```kubeadm init```, ```kubeadm join```

### 创建一个 Master 节点

```bash
kubeadm init --control-plane-endpoint=<ip> --pod-network-cidr=10.244.0.0/16 --image-repository registry.aliyuncs.com/google_containers
```

### 将一个 Node 节点加入到集群中

```bash
kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash> --control-plane 
```

## kubectl

kubectl 是 kubernetes 集群的命令行工具。

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

kubelet 是 master 派到 node 节点代表，管理本机容器:

- 一个集群中每个节点上运行的代理，它保证容器都运行在Pod中
- 负责维护容器的生命周期，同时也负责 Volume(CSI) 和 网络(CNI)的管理
