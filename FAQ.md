# FAQ

## [WARNING Swap]: swap is enabled; production deployments should disable swap unless testing the NodeSwap feature gate of the kubelet

- 临时禁用 swap:

    ```bash
    # swapoff -a

    # free -mh
    ```

    ***```swapon -a``` 开启 swap***

- 永久禁用 swap:

    ```bash
    # sed -i 's+/dev/mapper/centos-swap+#/dev/mapper/centos-swap+' /etc/fstab

    # reboot

    # free -mh
    ```

## Node节点处于NotReady的状态

从以下两个方面分析节点日志:

- ```kubectl describe nodes <node_name>```
- ```journalctl -f -u kubelet.service```

## [WARNING Hostname]: hostname "xxx" could not be reached

添加 hostname 映射:

```bash
# echo '127.0.0.1   xxx' >> /etc/hosts
```

## [WARNING Hostname]: hostname "xxx": lookup centos-docker-164 on 202.96.134.133:53: no such host

添加 hostname 映射:

```bash
# echo '127.0.0.1   xxx' >> /etc/hosts
```

## INSTALLATION FAILED: cannot re-use a name that is still in use

```bash
# helm ls --all-namespaces

# helm uninstall <NAME>
```

## failure: repodata/repomd.xml from kubernetes: [Errno 256] No more mirrors to try.

报错详情:

```
https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/repodata/repomd.xml: [Errno -1] repomd.xml signature could not be verified for kubernetes
```

修改 ```repo_gpgcheck=0```:

```bash
# sed -i 's+repo_gpgcheck=1+repo_gpgcheck=0+' /etc/yum.repos.d/kubernetes.repo
```

## 没有可用软件包 xx。

```bash
# yum install -y epel-release
```

## "command failed" err="failed to load kubelet config file, error: failed to load Kubelet config file /var/lib/kubelet/config.yaml

还需要执行 ```kubeadm init```，执行之后就会生成 ```/var/lib/kubelet/config.yaml```

## pod 无法删除，总是处于 terminate 状态

强制删除 pod:

```bash
# kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

批量强制删除 pod:

```bash
# kubectl delete pod $(kubectl get pod -n <namespace> | grep <pod-name> | awk '{print $1}') -n <namespace> --force --grace-period=0
```

## ingress 把前端的 http 请求转发给后台的 https 服务

```yml
ingress:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

## ksoftirqd/n 占用 cpu 过高

1. 查看设备造成的中断情况:
   ```bash
   cat /proc/interrupts
   ```
2. 针对中断数量高的网卡(如: ens33)，执行以一操作:
   ```bash
   ifdown ens33 && ifup ens33
   ```
