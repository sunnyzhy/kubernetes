# 使用 KubeKey 部署 kubernetes 集群

[kubekey](https://github.com/kubesphere/kubekey 'kubekey')

## 前言

本示例以 ```v3.0.7``` 版本为例。

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

## 部署

### 下载 kubekey

[kubekey-v3.0.7](https://github.com/kubesphere/kubekey/releases/download/v3.0.7/kubekey-v3.0.7-linux-amd64.tar.gz 'kubekey-v3.0.7')

### 创建 k8s 集群的模板文件

```bash
# tar -xzvf kubekey-v3.0.7-linux-amd64.tar.gz
kk

# ./kk create config --with-kubernetes v1.26.1 -f k8s.yaml

# ls
k8s.yaml  kk  kubekey-v3.0.7-linux-amd64.tar.gz
```

### 编辑 k8s 集群的模板文件

```bash
# vim k8s.yaml
```

```yml

apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: centos-docker-163, address: 192.168.5.163, internalAddress: 192.168.5.163, user: root, password: "X"}
  - {name: centos-docker-164, address: 192.168.5.164, internalAddress: 192.168.5.164, user: root, password: "X"}
  - {name: centos-docker-165, address: 192.168.5.165, internalAddress: 192.168.5.165, user: root, password: "X"}
  roleGroups:
    etcd:
    - centos-docker-163
    control-plane: 
    - centos-docker-163
    worker:
    - centos-docker-164
    - centos-docker-165
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.26.1
    clusterName: cluster.local
    autoRenewCerts: true
    containerManager: containerd
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []


```

配置说明:

- hosts
   - name: 主机名
   - address: 物理地址
   - internalAddress: 
   - user: 登录系统的用户名
   - password: 登录系统的密码
- roleGroups
   - etcd: 主机名
   - control-plane:  主节点
   - worker: 工作节点
- controlPlaneEndpoint
   - internalLoadbalancer: 负载均衡配置

### 创建 k8s 集群

```bash
./kk create cluster -f k8s.yaml
```

### 验证 k8s 集群

```bash
kubectl get nodes
```
