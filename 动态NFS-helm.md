# 动态 NFS - helm

## 部署 NFS 集群

- ***NFS 默认的配置文件: ```/etc/exports```***

- 准备三台物理机:

    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

### 安装 NFS 组件

在三台物理机上分别执行以下命令:

```bash
# yum install -y nfs-utils rpcbind
```

linux 系统默认安装的有 NFS 组件，可以忽略该步骤。

### 配置共享文件夹

在物理机 192.168.5.163(manager) 上执行以下命令:

```bash
# mkdir -p /nfs/data

# chmod -R 777 /nfs/data

# vim /etc/exports
```

```
/nfs/data *(rw,no_root_squash,sync)
```

配置生效:

```bash
# exportfs -r

# systemctl restart rpcbind && systemctl enable rpcbind

# systemctl restart nfs && systemctl enable nfs

# showmount -e 192.168.5.163
Export list for 192.168.5.163:
/nfs/data *
```

### 挂载集群节点

在物理机 192.168.5.164(worker) 和 192.168.5.165(worker) 上分别执行以下命令:

```bash
# mkdir -p /nfs/data

# chmod -R 777 /nfs/data

# mount 192.168.5.163:/nfs/data /nfs/data
```

### 测试是否挂载成功

在物理机 192.168.5.163(manager) 上执行以下命令:

```bash
# touch /nfs/data/1.log

# tree /nfs/data/
/nfs/data/
└── 1.log

0 directory, 1 files
```

在物理机 192.168.5.164(worker) 和 192.168.5.165(worker) 上分别执行以下命令:

```bash
# tree /nfs/data/
/nfs/data/
└── 1.log

0 directory, 1 files
```

在物理机 192.168.5.164(worker) 上执行以下命令:

```bash
# touch /nfs/data/2.data

# tree /nfs/data/
├── 1.log
└── 2.data

0 directories, 2 files
```

在物理机 192.168.5.163(manager) 和 192.168.5.165(worker) 上分别执行以下命令:

```bash
# tree /nfs/data/
├── 1.log
└── 2.data

0 directories, 2 files
```

## 创建动态 PV

[nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner 'nfs-subdir-external-provisioner')

### 创建命名空间

```bash
# kubectl create namespace iot
```

### 添加 chart 源

```bash
# helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

# helm repo update

# helm search repo nfs-subdir-external-provisioner
NAME                                              	CHART VERSION	APP VERSION	DESCRIPTION                                       
nfs-subdir-external-provisioner/nfs-subdir-exte...	4.0.16       	4.0.2      	nfs-subdir-external-provisioner is an automatic...
```

### 拉取 chart

```bash
# helm pull nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -d /usr/local/k8s/nfs-client-provisioner/
```

### 修改镜像

***```k8s.gcr.io``` 无法访问，所以需要把 ```repository``` 修改为可以访问的镜像仓库。***

```bash
# tar zxvf /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner-4.0.16.tgz -C /usr/local/k8s/nfs-client-provisioner

# sed -i 's+repository: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner+repository: registry.cn-hangzhou.aliyuncs.com/aliyun-zhy/nfs-subdir-external-provisioner+' /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner/values.yaml
```

### 重新制作 chart 包

```bash
# rm -rf /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner-4.0.16.tgz

# helm package /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner

# tree /usr/local/k8s/nfs-client-provisioner/
/usr/local/k8s/nfs-client-provisioner/
├── nfs-subdir-external-provisioner
│   ├── Chart.yaml
│   ├── ci
│   │   └── test-values.yaml
│   ├── README.md
│   ├── templates
│   │   ├── clusterrolebinding.yaml
│   │   ├── clusterrole.yaml
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── persistentvolumeclaim.yaml
│   │   ├── persistentvolume.yaml
│   │   ├── podsecuritypolicy.yaml
│   │   ├── rolebinding.yaml
│   │   ├── role.yaml
│   │   ├── serviceaccount.yaml
│   │   └── storageclass.yaml
│   └── values.yaml
└── nfs-subdir-external-provisioner-4.0.16.tgz
```

### 部署

- 部署方式 1:

    ```bash
    # helm install nfs-subdir-external-provisioner /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner-4.0.16.tgz  \
          --set nfs.server=192.168.5.163 \
          --set nfs.path=/nfs/data \
          --namespace iot
    ```

- 部署方式 2:

    ```bash
    # sed -i -e 's+server:+server: 192.168.5.163+' -e 's+path: /nfs-storage+path: /nfs/data+' /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner/values.yaml 

    # helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -f /usr/local/k8s/nfs-client-provisioner/nfs-subdir-external-provisioner/values.yaml -n iot
    ```

查看 deployment,pod:

```bash
# kubectl get deployment,pod -n iot
NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-subdir-external-provisioner   1/1     1            1           7m31s

NAME                                                   READY   STATUS      RESTARTS        AGE
pod/nfs-subdir-external-provisioner-8547f49d88-g2kvm   1/1     Running     0               7m31s
pod/test-pod                                           0/1     Completed   0               2m25s
```

### 测试

创建 test-claim:

```bash
# vim /usr/local/k8s/nfs-client-provisioner/test-claim.yaml
```

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  namespace: iot
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
  storageClassName: nfs-client
```

创建 test-pod:

```bash
# vim /usr/local/k8s/nfs-client-provisioner/test-pod.yaml
```

```yml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
  namespace: iot
spec:
  containers:
  - name: test-pod
    image: registry.aliyuncs.com/google_containers/busybox:1.24
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

测试 PVC:

```bash
# kubectl apply -f /usr/local/k8s/nfs-client-provisioner/test-claim.yaml
```

查看 PVC,PV 状态:

```bash
# kubectl get pvc,pv -n iot
NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
persistentvolumeclaim/test-claim             Bound    pvc-2175e0e3-53eb-4c93-9481-1fca4b5c9af0   1Mi        RWX            nfs-client          11s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                         STORAGECLASS        REASON   AGE
persistentvolume/pvc-2175e0e3-53eb-4c93-9481-1fca4b5c9af0   1Mi        RWX            Delete           Bound    iot/test-claim                nfs-client                   11s
```

查看 NFS Server 文件:

```bash
# tree /nfs/data
/nfs/data
└── iot-test-claim-pvc-2175e0e3-53eb-4c93-9481-1fca4b5c9af0

1 directory, 0 files
```

测试 Pod:

```bash
# kubectl apply -f /usr/local/k8s/nfs-client-provisioner/test-pod.yaml
```

查看 test-pod:

```bash
# kubectl get pods test-pod -n iot
NAME       READY   STATUS      RESTARTS   AGE
test-pod   0/1     Completed   0          23s
```

查看 NFS Server 文件，由 test-pod 创建的 ```SUCCESS``` 文件已挂载到了 ```/nfs/data``` 目录:

```bash
# tree /nfs/data
/nfs/data
└── iot-test-claim-pvc-2175e0e3-53eb-4c93-9481-1fca4b5c9af0
    └── SUCCESS

1 directory, 1 file
```

删除:

```bash
# kubectl delete -f /usr/local/k8s/nfs-client-provisioner/test-claim.yaml -f /usr/local/k8s/nfs-client-provisioner/test-pod.yaml
```

## FAQ

### mount.nfs: /nfs/data is busy or already mounted

```bash
umount -l /nfs/data

mount 192.168.5.163:/nfs/data /nfs/data
```
