# 卸载 kubernetes

```bash
kubeadm reset -f
rm -rvf $HOME/.kube
rm -rvf ~/.kube/
rm -rvf /etc/kubernetes/
rm -rvf /etc/systemd/system/kubelet.service.d
rm -rvf /etc/systemd/system/kubelet.service
rm -rvf /usr/bin/kube*
rm -rvf /etc/cni
rm -rvf /opt/cni
rm -rvf /var/lib/etcd
rm -rvf /var/etcd
yum  -y remove kube*
yum  -y remove docker*
```
