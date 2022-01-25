# 安装 kubernetes

## 1 安装 docker

[安装 docker](https://github.com/sunnyzhy/docker/blob/master/%E5%AE%89%E8%A3%85docker.md '安装 docker')

## 2 安装 kubernetes

### 2.1 安装 yum 源

[kubernetes下载地址](https://mirrors.aliyun.com/kubernetes/ 'kubernetes镜像')

```bash
# vim /etc/yum.repos.d/kubernates.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

### 2.2 更新缓存

```bash
# yum clean all

# yum -y makecache
```

### 2.3 查看 kubernetes 版本

```bash
# yum list | grep kubeadm
kubeadm.x86_64                            1.23.2-0                     kubernetes

# yum list kubelet --showduplicates | sort -r
```

### 2.4 安装 kubernetes

```bash
# yum install -y kubelet kubeadm kubectl
Installed:
  kubeadm.x86_64 0:1.23.2-0            kubectl.x86_64 0:1.23.2-0            kubelet.x86_64 0:1.23.2-0  

# rpm -ql kubelet
/etc/kubernetes/manifests
/etc/sysconfig/kubelet
/usr/bin/kubelet
/usr/lib/systemd/system/kubelet.service
```

### 2.5 测试 kubelet 服务(实际无需启动)

```bash
# systemctl start kubelet

# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; disabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Tue 2022-01-25 11:29:18 CST; 2s ago
     Docs: https://kubernetes.io/docs/
  Process: 7438 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
 Main PID: 7438 (code=exited, status=1/FAILURE)

Jan 25 11:29:18 node1 systemd[1]: Unit kubelet.service entered failed state.
Jan 25 11:29:18 node1 systemd[1]: kubelet.service failed.

# systemctl stop kubelet
```

### 2.6 初始化集群

```bash
# kubeadm init --apiserver-advertise-address=192.168.0.100 --kubernetes-version=v1.23.2 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.23.2
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING Swap]: swap is enabled; production deployments should disable swap unless testing the NodeSwap feature gate of the kubelet
	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```

逐一解决上述的 WARNING 和 ERROR:

- [WARNING Service-Docker]

   ```bash
   # systemctl enable docker.service
   ```

- [WARNING Swap]

   ```bash
   # vim /etc/sysconfig/kubelet
   KUBELET_EXTRA_ARGS="--fail-swap-on=false"
   
   # kubeadm init --apiserver-advertise-address=192.168.0.100 --kubernetes-version=v1.23.2 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=Swap
   ```

- [WARNING Service-Kubelet]

   ```bash
   # systemctl enable kubelet.service
   ```

- [ERROR DirAvailable--var-lib-etcd]

   ```bash
   # rm -rf /var/lib/etcd
   ```

参数说明:

- --apiserver-advertise-address: master 主机的 IP 地址

- --kubernetes-version: 当前安装的 kubernetes 的版本号

- --image-repository: 镜像地址，可以使用阿里云仓库地址: ```registry.aliyuncs.com/google_containers```

- --service-cidr: 默认 10.96.0.0/12, 无需更改，可以使用 ``kubeadm init --help``` 查看

- --pod-network-cidr: kubernetes 内部的 pod 节点之间网络可以使用的 IP 段，不能跟 service-cidr 重复，可以使用 ```10.244.0.0/16```
