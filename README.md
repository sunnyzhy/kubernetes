# Getting started with kubernetes

## Disable swap

```bash
sed -i 's+/dev/mapper/centos-swap+#/dev/mapper/centos-swap+' /etc/fstab

reboot
```

## Installing kubernetes

Download the ```k8s-\<VERSION\>.tar.gz``` archive from https://github.com/sunnyzhy/kubernetes/releases

- unzip file:
   ```bash
   tar  -xzvf k8s-\<VERSION\>.tar.gz
   cd k8s-\<VERSION\>
   chmod +x *.sh
   ```
- run ```./install-manager.sh <manager_ip>``` to install kubernetes manager:
   ```bash
   # ./install-manager.sh 192.168.0.100
   You can now join any number of control-plane nodes by copying certificate authorities
   and service account keys on each node and then running the following as root:

     kubeadm join 192.168.0.100:6443 --token rip8zm.q66uc0n4difhnp54 \
      --discovery-token-ca-cert-hash sha256:bef6b61859afc61d2a8cfbe14de99f891c34c37ae118e17716954707ecc11fac \
      --control-plane 

   Then you can join any number of worker nodes by running the following on each as root:

   kubeadm join 192.168.0.100:6443 --token rip8zm.q66uc0n4difhnp54 \
      --discovery-token-ca-cert-hash sha256:bef6b61859afc61d2a8cfbe14de99f891c34c37ae118e17716954707ecc11fac
   ```
- copy manager machine's ```/etc/kubernetes/admin.conf``` to worker machine's ```/etc/kubernetes/admin.conf```
- run ```./install-worker.sh <manager_ip:6443> <token> <discovery-token-ca-cert-hash>``` to install kubernetes worker:
   ```bash
   # ./install-worker.sh 192.168.0.100:6443 rip8zm.q66uc0n4difhnp54 sha256:bef6b61859afc61d2a8cfbe14de99f891c34c37ae118e17716954707ecc11fac
   ```

## Installing ingress-nginx

[ingress-nginx](./ingress-nginx%E5%AE%89%E8%A3%85.md 'ingress-nginx')

## Installing NFS

[NFS](./%E5%8A%A8%E6%80%81NFS-helm.md 'NFS')

## Supplement

### Default config files

#### containerd config file

```bash
# mkdir -p /etc/containerd

# containerd config default | tee /etc/containerd/config.toml

# cat /etc/containerd/config.toml
```

```grpc```'s  address default value is: ```/run/containerd/containerd.sock```

Set ```SystemdCgroup```:

```bash
sed -i 's+SystemdCgroup = false+SystemdCgroup = true+' /etc/containerd/config.toml
```

Replace image repository ```k8s.gcr.io``` with ```registry.aliyuncs.com/google_containers```:

```bash
sed -i 's+k8s.gcr.io/pause:3.6+registry.aliyuncs.com/google_containers/pause:3.6+' /etc/containerd/config.toml
```

#### kubeadm config file

```bash
# kubeadm config print init-defaults > /usr/local/kubeadm.yaml

# cat /usr/local/kubeadm.yaml
```

Set CRI:

```criSocket```'s default value is: ```unix:///var/run/containerd/containerd.sock```

```bash
sed -i 's+unix:\/\/\/var\/run\/containerd\/containerd.sock+unix:\/\/\/run\/containerd\/containerd.sock+' /usr/local/kubeadm.yaml
```

Replace image repository ```k8s.gcr.io``` with ```registry.aliyuncs.com/google_containers```:

```bash
sed -i 's+k8s.gcr.io+registry.aliyuncs.com\/google_containers+' /usr/local/kubeadm.yaml
```

### Change CR

#### kubelet use containerd

```bash
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7"
```

#### kubeadm use containerd

```bash
# kubeadm init --control-plane-endpoint=192.168.5.163 --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///run/containerd/containerd.sock --image-repository registry.aliyuncs.com/google_containers
```

OR

```bash
# kubeadm init --control-plane-endpoint=192.168.5.163 --pod-network-cidr=10.244.0.0/16 --config=/usr/local/kubeadm.yaml
```

### Kubeconfig file

```bash
# ls ~/.kube/config

# cat ~/.kube/config
```
