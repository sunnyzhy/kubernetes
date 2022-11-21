# containerd - 导入镜像

## 创建 docker 镜像

命令:

```bash
docker build -t <IMAGE_NAME>:<TAG> <DOCKERFILE_PATH>
```

示例:

```bash
docker build -t demo:1.0.0 /usr/local/app/demo/
```

## 给 docker 镜像创建标签

命令:

```bash
docker tag <IMAGE_NAME>:<TAG> core.harbor.domain/<PROJECT_NAME>/<IMAGE_NAME>:<TAG>
```

示例:

```bash
docker tag demo:1.0.0 core.harbor.domain/iot/demo:latest
```

## 导出 docker 镜像

命令:

```bash
docker save ccore.harbor.domain/<PROJECT_NAME>/<IMAGE_NAME>:<TAG> > <IMAGE_NAME>.tar
```

示例:

```bash
docker save core.harbor.domain/iot/demo:latest > demo.tar
```

## 把 docker 镜像传输到目标服务器

命令:

```bash
scp -r ./<IMAGE_NAME>.tar <TARGET_SERVER_USERNAME>@<TARGET_SERVER>:<TARGET_SERVER_PATH>
```

示例:

```bash
# scp -r ./demo.tar root@192.168.0.10:/usr/local/
root@192.168.0.10's password:
```

## 导入镜像

命令:

```bash
ctr -n k8s.io image import <TARGET_SERVER_PATH>/<IMAGE_NAME>.tar
```

示例:

```bash
ctr -n k8s.io image import /usr/local/demo.tar
```

## 查看导入的镜像

### ctr

命令:

```bash
ctr -n k8s.io image ls | grep <IMAGE_NAME>
```

示例:

```bash
ctr -n k8s.io image ls | grep demo
```

### crictl

命令:

```bash
crictl image ls | grep <IMAGE_NAME>
```

示例:

```bash
crictl image ls | grep demo
```
