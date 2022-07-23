# 动态 NFS

[nfs-client](https://github.com/kubernetes-retired/external-storage 'nfs-client')

## 创建 NFS 存储

***```/etc/exports``` 文件就是 NFS 默认的配置文件***

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

## 创建动态 PV

### 创建命名空间

```bash
# kubectl create namespace iot
```

### 创建 NFS 目录

```bash
# mkdir -p /usr/local/nfs
```

### 创建 rbac

```bash
# vim /usr/local/nfs/rbac.yaml
```

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: iot
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: iot
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: iot
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: iot
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: iot
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```bash
# kubectl apply -f /usr/local/nfs/rbac.yaml
```

### 创建 class

```bash
# vim /usr/local/nfs/class.yaml
```

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

```bash
# kubectl apply -f /usr/local/nfs/class.yaml
```

### 创建 deployment

```bash
# vim /usr/local/nfs/deployment.yaml
```

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: iot
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: gmoney23/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.5.163
            - name: NFS_PATH
              value: /nfs/data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.5.163
            path: /nfs/data
```

```bash
# kubectl apply -f /usr/local/nfs/deployment.yaml
```

不要使用源文件 deployment.yaml 里提供的镜像 ```quay.io/external_storage/nfs-client-provisioner:latest```，会报 pod 异常 ```selfLink was empty, can't make reference```。需要使用其他镜像，比如: ```gmoney23/nfs-client-provisioner:latest```，详情请参考 ```FAQ # PVC 一直处于 pending 状态```

### 测试

创建 test-claim:

```bash
# vim /usr/local/nfs/test-claim.yaml
```

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  namespace: iot
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

创建 test-pod:

```bash
# vim /usr/local/nfs/test-pod.yaml
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
    image: gcr.io/google_containers/busybox:1.24
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

测试:

```bash
# kubectl apply -f /usr/local/nfs/test-claim.yaml -f /usr/local/nfs/test-pod.yaml
```

删除:

```bash
# kubectl delete -f /usr/local/nfs/test-claim.yaml -f /usr/local/nfs/test-pod.yaml
```

## FAQ

### PVC 一直处于 pending 状态

pvc 异常 ```waiting for a volume to be created, either by external provisioner "fuseim.pri/ifs" or manually created by system administrator```:

```bash
# kubectl describe pvc test-claim -n iot
Events:
  Type    Reason                Age                     From                         Message
  ----    ------                ----                    ----                         -------
  Normal  ExternalProvisioning  7m49s (x5321 over 22h)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "fuseim.pri/ifs" or manually created by system administrator
```

pod 异常 ```selfLink was empty, can't make reference```:

```bash
# kubectl logs nfs-client-provisioner-58548f6578-4jngl -n iot
I0723 09:50:00.277312       1 controller.go:987] provision "iot/test-claim" class "managed-nfs-storage": started
E0723 09:50:00.283724       1 controller.go:1004] provision "iot/test-claim" class "managed-nfs-storage": unexpected error getting claim reference: selfLink was empty, can't make reference
```

注意事项:

1. 在 ```1.20 ~ 1.21``` 的 k8s 版本中，可以在 kube-apiserver.yaml 文件里增加以下属性解决：

    ```bash
    # vim /etc/kubernetes/manifests/kube-apiserver.yaml
    ```

    ```yml
    spec:
      containers:
      - command:
        - kube-apiserver
        ...
        - --feature-gates=RemoveSelfLink=false # 增加
    ```

    但是在 ```1.21`` 以上的 k8s 版本中已经移除了这个属性设置。如果依然要添加该属性，会造成 kube-apiserver 启动失败。

2. 正确的解决方法是在 deployment.yaml 里使用其他镜像，比如: ```gmoney23/nfs-client-provisioner:latest```

    ```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nfs-client-provisioner
      ...
    spec:
      ...
        spec:
          serviceAccountName: nfs-client-provisioner
          containers:
            - name: nfs-client-provisioner
              image: gmoney23/nfs-client-provisioner:latest
    ```
