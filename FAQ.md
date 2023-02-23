# FAQ

## 1. 0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) had untolerated taint {node.kubernetes.io/unreachable: }

服务器内存不足或磁盘空间不足引起的。

解决方法：清理服务器内存或磁盘空间。

## 2. [WARNING Swap]: swap is enabled; production deployments should disable swap unless testing the NodeSwap feature gate of the kubelet

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

## 3. Node节点处于NotReady的状态

从以下两个方面分析节点日志:

- ```kubectl describe nodes <node_name>```
- ```journalctl -f -u kubelet.service```

## 4. [WARNING Hostname]: hostname "xxx" could not be reached

添加 hostname 映射:

```bash
# echo '127.0.0.1   xxx' >> /etc/hosts
```

## 5. [WARNING Hostname]: hostname "xxx": lookup centos-docker-164 on 202.96.134.133:53: no such host

添加 hostname 映射:

```bash
# echo '127.0.0.1   xxx' >> /etc/hosts
```

## 6. INSTALLATION FAILED: cannot re-use a name that is still in use

1. 使用 ```helm -n <namespace> ls -a``` 查询全部，可以看到失败的服务
2. 使用 ```helm -n <namespace> delete <packagename>``` 删除失败的服务
   或使用 ```helm -n <namespace> delete $(helm -n <namespace> ls -a | awk 'NR>1 {print $1}')``` 批量删除失败的服务
3. 使用 ```helm install``` 重新安装服务。

## 7. failure: repodata/repomd.xml from kubernetes: [Errno 256] No more mirrors to try.

报错详情:

```
https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/repodata/repomd.xml: [Errno -1] repomd.xml signature could not be verified for kubernetes
```

修改 ```repo_gpgcheck=0```:

```bash
# sed -i 's+repo_gpgcheck=1+repo_gpgcheck=0+' /etc/yum.repos.d/kubernetes.repo
```

## 8. 没有可用软件包 xx。

```bash
# yum install -y epel-release
```

## 9. "command failed" err="failed to load kubelet config file, error: failed to load Kubelet config file /var/lib/kubelet/config.yaml

还需要执行 ```kubeadm init```，执行之后就会生成 ```/var/lib/kubelet/config.yaml```

## 10. pod 无法删除，总是处于 terminate 状态

强制删除 pod:

```bash
# kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

批量强制删除 pod:

```bash
# kubectl delete pod $(kubectl get pod -n <namespace> | grep <pod-name> | awk 'NR>1{print $1}') -n <namespace> --force --grace-period=0
```

## 11. ingress 把前端的 http 请求转发给后台的 https 服务

```yml
ingress:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

## 12. ksoftirqd/n 占用 cpu 过高

1. 查看设备造成的中断情况:
   ```bash
   cat /proc/interrupts
   ```
2. 针对中断数量高的网卡(如: ens33)，执行以一操作:
   ```bash
   ifdown ens33 && ifup ens33
   ```

## 13. 如何删除 pod 缓存的镜像

1. 查看 pod 所在的工作节点:
   ```bash
   # kubectl describe pod nginx-0 -n iot | grep Node:
   Node:             centos-docker-165/192.168.5.165
   ```
2. 查看 pod 的容器镜像:
   ```bash
   # kubectl describe pod nginx-0 -n iot | grep -A 4 Containers:
   Containers:
     nginx:
       Container ID:   containerd://85b25144ac12ecc0790981ce072865521e6801b4b4debf2311e4c6a878b4b9c7
       Image:          core.harbor.domain/iot/nginx:latest
       Image ID:       core.harbor.domain/iot/nginx@sha256:55f8523e6a3a7200f9b5c74a881075161aaf68442fc5f802fe3d55f059ced390
   ```
3. 转到工作节点 ```192.168.5.165``` 查看镜像:
   ```bash
   # ctr -n k8s.io image ls | grep nginx | awk '{print $1}'
   core.harbor.domain/iot/nginx:latest
   core.harbor.domain/iot/nginx@sha256:55f8523e6a3a7200f9b5c74a881075161aaf68442fc5f802fe3d55f059ced390
   ```
4. 删除镜像:
   ```bash
   ctr -n k8s.io image remove $(ctr -n k8s.io image ls | grep nginx | awk '{print $1}')
   ```

## 14. etcd pod 启动失败

问题描述:

```
"Error syncing pod, skipping" err="failed to \"StartContainer\" for \"etcd\" with CrashLoopBackOff: \"back-off 40s restarting failed container=etcd pod=etcd-node-1_kube-system(2081d4e5d93b83634021b6ef0c97cca5)\"" pod="kube-system/etcd-node-1" podUID=2081d4e5d93b83634021b6ef0c97cca5
```

出现这种情况，大概率是因为集群节点遇到突然断电或者 ```init 0``` 等非正常关机而引起的 ```etcd``` 数据被损坏。

解决方法:

1. 在主节点删除损坏的 ```etcd``` 数据，相当于集群重置:
   ```bash
   rm -rf /var/lib/etcd/member/*
   
   reboot
   ```
   
   重新生成集群 ```token``` :
   ```bash
   # kubeadm token create --print-join-command
   kubeadm join 20.0.0.101:6443 --token isw8x4.cc5xq90713gsi8o3 --discovery-token-ca-cert-hash sha256:e4085e3cdfbb73ed6c31580eff2bdda3f00cdf4153a3c4fda831da2298443881 
   ```
   
   在工作节点上执行以下命令把工作节点加入集群:
   ```bash
   kubeadm reset
   
   kubeadm join 20.0.0.101:6443 --token isw8x4.cc5xq90713gsi8o3 --discovery-token-ca-cert-hash sha256:e4085e3cdfbb73ed6c31580eff2bdda3f00cdf4153a3c4fda831da2298443881 
   ```
   
2. 【推荐】定时备份 ```etcd``` 数据，以备数据故障时还原
   - 备份脚本
      ```bash
      # vim /root/etcd_bak.sh
      #/bin/sh
      mkdir -p /root/bak/etcd
      cp -r /var/lib/etcd/member /root/bak/etcd
      
      # chmod +x /root/etcd_bak.sh
      ```
   - 创建定时任务(每隔1小时执行一次)
      ```bash
      # crontab -e
      */60 * * * * /root/etcd_bak.sh
      ```
