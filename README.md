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

```criSocket```'s default value is: ```unix:///var/run/containerd/containerd.sock```

Replace image repository ```k8s.gcr.io``` with ```registry.aliyuncs.com/google_containers```:

```bash
sed -i 's+k8s.gcr.io/pause:3.6+registry.aliyuncs.com/google_containers/pause:3.6+' /usr/local/kubeadm.yaml
```
