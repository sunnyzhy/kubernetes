# 安装 helm

[helm](https://github.com/helm/helm/releases 'helm')

## 安装 helm

```bash
# wget -c https://get.helm.sh/helm-v3.9.1-linux-amd64.tar.gz -P /usr/local/

# tar Cxzvf /usr/local /usr/local/helm-v3.9.1-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md

# mv /usr/local/linux-amd64/helm /usr/local/bin/

# rm -rf /usr/local/linux-amd64
```

## 配置 chart 仓库

### 添加阿里云的 chart 仓库

```bash
# helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

### 添加 bitnami 的 chart 仓库

```bash
# helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 更新 chart 仓库

```bash
# helm repo update

# helm repo list
NAME   	URL                                                   
aliyun 	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
bitnami	https://charts.bitnami.com/bitnami   
```

```bash
# helm repo remove <NAME>
```
