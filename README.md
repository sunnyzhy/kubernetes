# Getting started with kubernetes

## Installing kubernetes

Download the k8s-\<VERSION\>.tar.gz archive from https://github.com/sunnyzhy/kubernetes/releases
  
- run ```./k8s.sh``` to install kubernetes
- run ```./k8s-cluster.sh <master_ip>``` to install kubernetes cluster

   master_ip is your master machine ip. such as ```./k8s-cluster.sh 192.168.0.100```

## Default config files

### containerd config file

```bash
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

### kubeadm config file

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

## Change CR

### kubelet use containerd

```bash
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7"
```

### kubeadm use containerd

```bash
# kubeadm init --control-plane-endpoint=192.168.5.163 --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///run/containerd/containerd.sock --image-repository registry.aliyuncs.com/google_containers
```

or

```bash
# kubeadm init --control-plane-endpoint=192.168.5.163 --pod-network-cidr=10.244.0.0/16 --config=/usr/local/kubeadm.yaml
```
