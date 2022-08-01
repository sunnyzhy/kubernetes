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
