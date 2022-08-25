# helm 命令

## 添加 chart 源

```bash
helm repo add <repo_name> <repo_chart_url>

helm repo update
```

示例:

```bash
helm repo add emqx https://repos.emqx.io/charts

helm repo update
```

## 查找 chart 源

```bash
helm search repo <repo_name>
```

示例:

```bash
helm search repo nginx
```

## 拉取 chart

```bash
# helm pull <chart_name> -d <local_dir>
```

示例:

```bash
helm pull bitnami/nginx -d /usr/local/k8s/nginx
```

## 打包本地 chart

```bash
helm package <local_chart_dir> -d <local_dir>
```

示例:

```bash
helm package /usr/local/k8s/nginx/nginx -d /usr/local/k8s/nginx
```

## 部署 chart

### 部署远程 chart

```bash
helm install <release_name> <chart_name> -n <namespace>
```

示例:

```bash
helm install my-nginx bitnami/nginx -n dev
```

***OR***

```bash
helm install <release_name> <chart_url.tgz> -n <namespace>
```

示例:

```bash
helm install my-nginx https://example.com/charts/nginx-13.1.7.tgz -n dev
```

### 部署本地 chart

```bash
helm install <release_name> <local_chart.tgz> -n <namespace>
```

示例:

```bash
helm install my-nginx ./nginx/nginx-13.1.7.tgz -n dev
```

***OR***

```bash
helm install <release_name> -f <resource.yaml> <local_chart_dir> -n <namespace>
```

示例:

```bash
helm install my-nginx -f my-resource.yaml  ./nginx -n dev
```
