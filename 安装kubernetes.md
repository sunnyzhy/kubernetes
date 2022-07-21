# 安装 kubernetes

## 前言

***基于 containerd 的容器运行时部署 k8s 集群。***

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|control-plane(manager)|
    |192.168.5.164|centos-docker-164|<none>(worker)|
    |192.168.5.165|centos-docker-165|<none>(worker)|

- 三台物理机都需要安装
   - ```Container Runtimes```
   - ```kubelet```
   - ```kubeadm```
   - ```kubectl```

## 安装 Container Runtimes

### 安装必备环境

```bash
# vim /etc/modules-load.d/k8s.conf
```

```conf
overlay
br_netfilter
```

```bash
# modprobe overlay

# modprobe br_netfilter

# vim /etc/sysctl.d/k8s.conf
```

```conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```

```bash
# sysctl --system

# yum install -y grubby

# grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
```

### 安装 containerd

[containerd](https://github.com/containerd/containerd/releases 'containerd')

```bash
# wget -c https://github.com/containerd/containerd/releases/download/v1.6.6/containerd-1.6.6-linux-amd64.tar.gz -P /usr/local/

# tar Cxzvf /usr/local /usr/local/containerd-1.6.6-linux-amd64.tar.gz
bin/
bin/containerd-shim
bin/containerd
bin/containerd-shim-runc-v1
bin/containerd-stress
bin/containerd-shim-runc-v2
bin/ctr
```

### 安装 systemd

[containerd.service](https://github.com/containerd/containerd/blob/main/containerd.service 'containerd.service')

```bash
# mkdir -p /usr/local/lib/systemd/system

# vim /usr/local/lib/systemd/system/containerd.service
```

由于 ```raw#containerd.service``` 无法下载，所以贴出 ```containerd.service``` 源文件内容:

```service
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

```bash
# chmod +x /usr/local/lib/systemd/system/containerd.service

# systemctl daemon-reload

# systemctl enable --now containerd
```

### 安装 runc

[runc](https://github.com/opencontainers/runc/releases 'runc')

```bash
# wget -c https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64 -P /usr/local/

# install -m 755 /usr/local/runc.amd64 /usr/local/sbin/runc
```

### 安装 CNI plugins

[CNI plugins](https://github.com/containernetworking/plugins/releases 'CNI plugins')

```bash
# wget -c https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz -P /usr/local/

# mkdir -p /opt/cni/bin

# tar Cxzvf /opt/cni/bin /usr/local/cni-plugins-linux-amd64-v1.1.1.tgz
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

### 配置 systemd cgroup driver

```bash
# containerd config default | tee /etc/containerd/config.toml

# sed -i 's+SystemdCgroup = false+SystemdCgroup = true+' /etc/containerd/config.toml

# sed -i 's+k8s.gcr.io/pause:3.6+registry.aliyuncs.com/google_containers/pause:3.6+' /etc/containerd/config.toml

# systemctl restart containerd
```

### 查看 containerd

```bash
# crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a | grep kube | grep -v pause
9a727650c613b       2ae1ba6417cbc       3 minutes ago       Running             kube-proxy                0                   f372f7f76d220       kube-proxy-v9lk2
cc0c0a312d5fd       3a5aa3a515f5d       3 minutes ago       Running             kube-scheduler            0                   57ea5834c6733       kube-scheduler-centos-docker-163
29370743dd9b4       d521dd763e2e3       3 minutes ago       Running             kube-apiserver            0                   dbf69505210f0       kube-apiserver-centos-docker-163
8419adae1fc6b       586c112956dfc       3 minutes ago       Running             kube-controller-manager   0                   ef93d9aef0c2f       kube-controller-manager-centos-docker-163
```

## 安装 kubernetes

### 配置 kubernetes 的 yum 源

[kubernetes 阿里云镜像](https://developer.aliyun.com/mirror/kubernetes/ 'kubernetes 阿里云镜像')

```bash
# vim /etc/yum.repos.d/kubernetes.repo
```

```repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

### 安装 kubernetes

```bash
# setenforce 0

# sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# yum install -y --nogpgcheck kubelet kubeadm kubectl --disableexcludes=kubernetes

# systemctl enable --now kubelet

# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since 一 2022-07-18 18:23:17 CST; 14h ago
     Docs: https://kubernetes.io/docs/
 Main PID: 25434 (kubelet)
    Tasks: 17
   Memory: 40.7M
   CGroup: /system.slice/kubelet.service
           └─25434 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc...

7月 18 18:25:47 centos-docker-163 kubelet[25434]: E0718 18:25:47.284750   25434 kuberuntime_sandbox.go:70] "Failed...
7月 18 18:25:47 centos-docker-163 kubelet[25434]: E0718 18:25:47.284776   25434 kuberuntime_manager.go:815] "Creat...
7月 18 18:25:47 centos-docker-163 kubelet[25434]: E0718 18:25:47.284888   25434 pod_workers.go:951] "Error syn...8_ku
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.591641   25434 topology_manager.go:200] "Topo...ler"
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.636494   25434 reconciler.go:270] "operationExecu...
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.636561   25434 reconciler.go:270] "operationExecu...
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.636607   25434 reconciler.go:270] "operationExecu...
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.636648   25434 reconciler.go:270] "operationExecu...
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.636688   25434 reconciler.go:270] "operationExecu...
7月 18 18:25:53 centos-docker-163 kubelet[25434]: I0718 18:25:53.636729   25434 reconciler.go:270] "operationExecu...
Hint: Some lines were ellipsized, use -l to show in full.

# rpm -ql kubelet
/etc/kubernetes/manifests
/etc/sysconfig/kubelet
/usr/bin/kubelet
/usr/lib/systemd/system/kubelet.service
```

查看 kubelet 详情:

```bash
# systemctl status kubelet.service -l | grep /usr/bin/kubelet | sed 's/[0-9]\{3,\} \/usr\/bin\/kubelet/ \/usr\/bin\/kubelet/g'
           └─ /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remote --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7
```

由于使用的 CR 是 containerd，所以 kubelet 服务跟 container 相关的参数如下:

- --container-runtime=remote

- --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock

- --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7

查看 cgroupDriver 详情:

```bash
# cat /var/lib/kubelet/config.yaml | grep  cgroupDriver
cgroupDriver: systemd
```

由于使用的 CR 是 containerd，所以 cgroupDriver 为 ```systemd```

### 更新缓存

```bash
# yum clean all

# yum -y makecache
```

### 初始化集群

参数说明:

- --control-plane-endpoint: master 主机的 IP 地址

- --pod-network-cidr=10.244.0.0/16: 集群的cidr，可以配置为 Flannel 网络模式: ```10.244.0.0/16```

- --kubernetes-version: 当前安装的 kubernetes 的版本号，可以不写，默认会自动获取版本

- --cri-socket: 容器运行时接口，在初始化集群的时候，会自动获取已安装的容器运行时；可以不写，默认为: ```unix:///var/run/containerd/containerd.sock```

- --image-repository: 镜像地址，可以使用阿里云仓库地址: ```registry.aliyuncs.com/google_containers```

- 其他参数为可选参数

```bash
# kubeadm init --control-plane-endpoint=192.168.5.163 --pod-network-cidr=10.244.0.0/16 --image-repository registry.aliyuncs.com/google_containers
[init] Using Kubernetes version: v1.24.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [centos-docker-163 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.5.163]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [centos-docker-163 localhost] and IPs [192.168.5.163 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [centos-docker-163 localhost] and IPs [192.168.5.163 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 8.504141 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node centos-docker-163 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node centos-docker-163 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: lc81r0.ts58t03upj136xd9
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.5.163:6443 --token lc81r0.ts58t03upj136xd9 \
	--discovery-token-ca-cert-hash sha256:af924c26c5b40bf4f7df8a8a396bf7904e592cde8ffc1f8bb1cece01e4f9f352 \
	--control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.5.163:6443 --token lc81r0.ts58t03upj136xd9 \
	--discovery-token-ca-cert-hash sha256:af924c26c5b40bf4f7df8a8a396bf7904e592cde8ffc1f8bb1cece01e4f9f352 
```

提示 ```Your Kubernetes control-plane has initialized successfully!``` 说明集群初始化成功。

### 授权

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

根据 ```初始化集群``` 步骤里的提示，执行授权:

```bash
# mkdir -p $HOME/.kube

# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装 pod network

```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
```

根据 ```初始化集群``` 步骤里的提示，安装 pod network:

1. 先下载:
[flannel](https://github.com/flannel-io/flannel/releases 'flannel')

```bash
# wget -c https://github.com/flannel-io/flannel/releases/download/v0.18.1/flannel-v0.18.1-linux-amd64.tar.gz -P /usr/local/

# mkdir -p /opt/bin/flanneld

# tar Cxzvf /opt/bin/flanneld /usr/local/flannel-v0.18.1-linux-amd64.tar.gz
flanneld
mk-docker-opts.sh
README.md
```

2. 安装 pod network:
[kube-flannel.yml](https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml 'kube-flannel.yml')

```bash
# vim /usr/local/kube-flannel.yml
```

由于 ```raw#kube-flannel.yml``` 无法下载，所以贴出 ```kube-flannel.yml``` 源文件内容:

```yml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.18.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel:v0.18.1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.18.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel:v0.18.1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

```bash
# kubectl apply -f /usr/local/kube-flannel.yml
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

### 查看节点状态

节点的正常状态如下:

- ```control-plane``` 状态为 ```Ready```
- 命名空间下的各个 ```pod``` 状态为 ```Running```
   
   当有节点加入集群之后，```ContainerCreating``` 会变为 ```Running```

在管理节点 192.168.5.163 查看节点状态:

```bash
# kubectl get nodes
NAME                STATUS   ROLES           AGE     VERSION
centos-docker-163   Ready    control-plane   4m41s   v1.24.3

# kubectl get pods --all-namespaces
NAMESPACE      NAME                                        READY   STATUS              RESTARTS   AGE
kube-flannel   kube-flannel-ds-dlz87                       1/1     Running             0          6s
kube-system    coredns-74586cf9b6-lvf9f                    0/1     ContainerCreating   0          2m27s
kube-system    coredns-74586cf9b6-xpmz8                    0/1     ContainerCreating   0          2m27s
kube-system    etcd-centos-docker-163                      1/1     Running             2          2m41s
kube-system    kube-apiserver-centos-docker-163            1/1     Running             2          2m41s
kube-system    kube-controller-manager-centos-docker-163   1/1     Running             0          2m44s
kube-system    kube-proxy-ns6rn                            1/1     Running             0          2m27s
kube-system    kube-scheduler-centos-docker-163            1/1     Running             2          2m41s
```

### 添加新节点到集群

在 192.168.5.164 上执行以下命令:

```bash
# kubeadm join 192.168.5.163:6443 --token lc81r0.ts58t03upj136xd9 --discovery-token-ca-cert-hash sha256:af924c26c5b40bf4f7df8a8a396bf7904e592cde8ffc1f8bb1cece01e4f9f352
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

在 192.168.5.165 上执行以下命令:

```bash
# kubeadm join 192.168.5.163:6443 --token lc81r0.ts58t03upj136xd9 --discovery-token-ca-cert-hash sha256:af924c26c5b40bf4f7df8a8a396bf7904e592cde8ffc1f8bb1cece01e4f9f352
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

在管理节点 192.168.5.163 查看节点状态:

```bash
# kubectl get nodes
NAME                STATUS   ROLES           AGE     VERSION
centos-docker-163   Ready    control-plane   17h     v1.24.3
centos-docker-164   Ready    <none>          91m     v1.24.3
centos-docker-165   Ready    <none>          2m25s   v1.24.3

# kubectl get pods --all-namespaces
NAMESPACE      NAME                                        READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-dlz87                       1/1     Running   0          17h
kube-flannel   kube-flannel-ds-jvjxf                       1/1     Running   1          91m
kube-flannel   kube-flannel-ds-qnpl8                       1/1     Running   0          2m44s
kube-system    coredns-74586cf9b6-lvf9f                    1/1     Running   0          17h
kube-system    coredns-74586cf9b6-xpmz8                    1/1     Running   0          17h
kube-system    etcd-centos-docker-163                      1/1     Running   2          17h
kube-system    kube-apiserver-centos-docker-163            1/1     Running   2          17h
kube-system    kube-controller-manager-centos-docker-163   1/1     Running   0          17h
kube-system    kube-proxy-kdzkc                            1/1     Running   0          2m44s
kube-system    kube-proxy-nrrn2                            1/1     Running   0          44m
kube-system    kube-proxy-xl7wm                            1/1     Running   0          44m
kube-system    kube-scheduler-centos-docker-163            1/1     Running   2          17h
```

### 配置 ipvs 模式

1. 在```管理节点/工作节点```上查看模式:

   ```bash
   # curl 127.0.0.1:10249/proxyMode
   iptables
   ```

2. 在```管理节点```上开启 ipvs 模式:

   ```bash
   # kubectl edit cm kube-proxy -n kube-system
   ```

   ***把 ```mode: ""``` 修改为 ```mode: "ipvs"```***

3. 在```管理节点```上删除当前的 kube-proxy:

   ```bash
   # kubectl get pod -n kube-system|grep kube-proxy|awk '{system("kubectl delete pod "$1" -n kube-system")}'
   ```

   ***删除后 ```kube-proxy``` 会自动重新生成，重新生成后即采用 ```ipvs ``` 模式，并且作用于 k8s 集群所有的节点。***

4. 在```管理节点/工作节点```上查看模式:

   ```bash
   # curl 127.0.0.1:10249/proxyMode
   ipvs
   ```
