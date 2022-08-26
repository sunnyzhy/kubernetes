# helm 命令

## 添加源

```bash
helm repo add emqx https://repos.emqx.io/charts

helm repo update
```

## 查找源

```bash
helm search repo nginx
```

## 拉取

```bash
helm pull bitnami/nginx -d /usr/local/k8s/nginx
```

## 打包

```bash
helm package /usr/local/k8s/nginx/nginx -d /usr/local/k8s/nginx
```

## 部署

### 从远程部署

```bash
helm install my-nginx bitnami/nginx -n dev
```

***OR***

```bash
helm install my-nginx https://example.com/charts/nginx-13.1.7.tgz -n dev
```

### 从本地部署

```bash
helm install my-nginx ./nginx/nginx-13.1.7.tgz -n dev
```

***OR***

```bash
helm install my-nginx -f my-resource.yaml  ./nginx -n dev
```

## 更新

```bash
helm upgrade my-nginx ./nginx/nginx-13.1.7.tgz -n dev
```

***OR***

```bash
helm upgrade my-nginx -f my-resource.yaml  ./nginx -n dev
```

## 删除

```bash
helm uninstall my-nginx -n dev
```

***OR***

```bash
helm uninstall my-nginx -n dev
```

## 自定义 chart

```bash
helm create my-nginx
```
