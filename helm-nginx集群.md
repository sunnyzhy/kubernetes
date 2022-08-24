# helm - nginx 集群

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

## 创建 nginx 目录

```bash
# mkdir -p /usr/local/k8s/nginx
```

## 查找 chart

```bash
# helm search repo nginx
NAME                            	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/nginx-ingress            	0.9.5        	0.10.2     	An nginx Ingress controller that uses ConfigMap...
aliyun/nginx-lego               	0.3.1        	           	Chart for nginx-ingress-controller and kube-lego  
bitnami/nginx                   	13.1.7       	1.23.1     	NGINX Open Source is a web server that can be a...
bitnami/nginx-ingress-controller	9.2.27       	1.3.0      	NGINX Ingress Controller is an Ingress controll...
bitnami/nginx-intel             	2.0.16       	0.4.7      	NGINX Open Source for Intel is a lightweight se...
nginx/ingress-nginx             	4.2.1        	1.3.0      	Ingress controller for Kubernetes using NGINX a...
bitnami/kong                    	5.0.2        	2.7.0      	Kong is a scalable, open source API layer (aka ...
aliyun/gcloud-endpoints         	0.1.0        	           	Develop, deploy, protect and monitor your APIs ...
```

## 拉取 chart

```bash
# helm pull bitnami/nginx -d /usr/local/k8s/nginx
```

## 修改 chart 配置

### 修改 values.yaml

```bash
# tar zxvf /usr/local/k8s/nginx/nginx-13.1.7.tgz -C /usr/local/k8s/nginx

# vim /usr/local/k8s/nginx/nginx/values.yaml
```

```yml
extraEnvVars:
  - name: TZ
    value: Asia/Shanghai

replicaCount: 2

serverBlock:  |-
  server {
    listen 0.0.0.0:8080;
    root /app;
    location / {
      index index.html index.php;
    }

    location /baidu {
      proxy_pass https://www.baidu.com/;
    }
  }

staticSitePVC: "nginx-data"

persistence:
  storageClass: nfs-client
  accessMode: ReadWriteMany
  size: 10Gi

ingress:
  enabled: true
  hostname: yun.cn
  ingressClassName: "nginx"

service:
  type: ClusterIP
  clusterIP: None
```

### 部署 PVC

```bash
# vim /usr/local/k8s/nginx/nginx/templates/pvc.yaml
```

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: {{ .Values.staticSitePVC }}
  namespace: {{ include "common.names.namespace" . | quote }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
  storageClassName: {{ .Values.persistence.storageClass }}
```

### 部署对外的 service

在 ```svc.yaml``` 里添加 NodePort 类型的 Service，对外提供服务（非必需，功能同 ingress）:

```bash
# vim /usr/local/k8s/nginx/nginx/templates/svc.yaml
```

```yml

---
apiVersion: v1
kind: Service
metadata:
  name: {{ template "common.names.fullname" . }}-service
  namespace: {{ include "common.names.namespace" . | quote }}
  labels: {{- include "common.labels.standard" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
spec:
  selector: {{- include "common.labels.matchLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.ports.http }}
      targetPort: {{ .Values.service.targetPort.http }}
      {{- if and (or (eq .Values.service.type "NodePort") (eq .Values.service.type "LoadBalancer")) (not (empty .Values.service.nodePorts.http)) }}
      nodePort: {{ .Values.service.nodePorts.http }}
      {{- end }}
      nodePort: 30088
  type: NodePort
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/nginx/nginx-13.1.7.tgz

# helm package /usr/local/k8s/nginx/nginx -d /usr/local/k8s/nginx

# tree /usr/local/k8s/nginx
/usr/local/k8s/nginx
├── nginx
│   ├── Chart.lock
│   ├── charts
│   │   └── common
│   │       ├── Chart.yaml
│   │       ├── README.md
│   │       ├── templates
│   │       │   ├── _affinities.tpl
│   │       │   ├── _capabilities.tpl
│   │       │   ├── _errors.tpl
│   │       │   ├── _images.tpl
│   │       │   ├── _ingress.tpl
│   │       │   ├── _labels.tpl
│   │       │   ├── _names.tpl
│   │       │   ├── _secrets.tpl
│   │       │   ├── _storage.tpl
│   │       │   ├── _tplvalues.tpl
│   │       │   ├── _utils.tpl
│   │       │   ├── validations
│   │       │   │   ├── _cassandra.tpl
│   │       │   │   ├── _mariadb.tpl
│   │       │   │   ├── _mongodb.tpl
│   │       │   │   ├── _mysql.tpl
│   │       │   │   ├── _postgresql.tpl
│   │       │   │   ├── _redis.tpl
│   │       │   │   └── _validations.tpl
│   │       │   └── _warnings.tpl
│   │       └── values.yaml
│   ├── Chart.yaml
│   ├── README.md
│   ├── templates
│   │   ├── deployment.yaml
│   │   ├── extra-list.yaml
│   │   ├── health-ingress.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   ├── pdb.yaml
│   │   ├── prometheusrules.yaml
│   │   ├── pvc.yaml
│   │   ├── server-block-configmap.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── servicemonitor.yaml
│   │   ├── svc.yaml
│   │   └── tls-secrets.yaml
│   ├── values.schema.json
│   └── values.yaml
└── nginx-13.1.7.tgz
```

## 部署

```bash
# helm install nginx-cluster /usr/local/k8s/nginx/nginx-13.1.7.tgz -n iot
NAME: nginx-cluster
LAST DEPLOYED: Thu Aug 18 17:28:37 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 13.1.7
APP VERSION: 1.23.1

** Please be patient while the chart is being deployed **
NGINX can be accessed through the following DNS name from within your cluster:

    nginx-cluster.iot.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace iot -w nginx-cluster'

    export SERVICE_PORT=$(kubectl get --namespace iot -o jsonpath="{.spec.ports[0].port}" services nginx-cluster)
    export SERVICE_IP=$(kubectl get svc --namespace iot nginx-cluster -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo "http://${SERVICE_IP}:${SERVICE_PORT}"
```

查看资源:

```bash
# kubectl get deploy -n iot | grep nginx
nginx-cluster                     2/2     2            2           22s

# kubectl get pod -n iot -o wide | grep nginx
nginx-cluster-7cd848b99f-b7wcn                     1/1     Running   0          12s   10.244.1.100   centos-docker-164   <none>           <none>
nginx-cluster-7cd848b99f-ngw7v                     1/1     Running   0          12s   10.244.2.92    centos-docker-165   <none>           <none>

# kubectl get pvc,pv -n iot | grep nginx
persistentvolumeclaim/nginx-data   Bound    pvc-f975d723-88d2-4c94-932a-ab6947af6415   10Gi       RWX            nfs-client     26s
persistentvolume/pvc-f975d723-88d2-4c94-932a-ab6947af6415   10Gi       RWX            Delete           Bound    iot/nginx-data   nfs-client              26s

# kubectl get svc -n iot | grep nginx
nginx-cluster           ClusterIP   None           <none>        80/TCP         60s
nginx-cluster-service   NodePort    10.100.63.17   <none>        80:30088/TCP   60s

# kubectl get ingress -n iot | grep nginx
nginx-cluster   nginx   yun.cn   10.102.1.248   80      81s
```

部署 nginx  的 ```index.html```:

```bash
# vim /nfs/data/iot-nginx-data-pvc-f975d723-88d2-4c94-932a-ab6947af6415/index.html
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

```bash
# vim /nfs/data/iot-nginx-data-pvc-f975d723-88d2-4c94-932a-ab6947af6415/50x.html
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Error</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>An error occurred.</h1>
<p>Sorry, the page you are looking for is currently unavailable.<br/>
Please try again later.</p>
<p>If you are the system administrator of this resource then you should check
the error log for details.</p>
<p><em>Faithfully yours, nginx.</em></p>
</body>
</html>
```

## 内部访问 nginx 集群

```bash
# curl 10.100.63.17
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# curl 10.100.63.17/baidu
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=https://ss1.bdstatic.com/5eN1bjq8AAUYm2zgoY3K/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus=autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn" autofocus></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=https://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');
                </script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

## 外部访问 nginx 集群

### 通过对外 Service 的方式

外部服务器访问 nginx:

```bash
# curl 192.168.5.163:30088
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# curl 192.168.5.163:30088/baidu
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=https://ss1.bdstatic.com/5eN1bjq8AAUYm2zgoY3K/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus=autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn" autofocus></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=https://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');
                </script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

### 通过 ingress 的方式

在任一外部服务器的 hosts 文件里配置域名映射:

```bash
# vim /etc/hosts
192.168.5.165 yun.cn
```

外部服务器访问 nginx:

```bash
# curl yun.cn
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# curl yun.cn/baidu
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=https://ss1.bdstatic.com/5eN1bjq8AAUYm2zgoY3K/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus=autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn" autofocus></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=https://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');
                </script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

## FAQ

### 403 Forbidden

原因: nfs 挂载的 nginx 目录缺少 ```index.html```

解决方法: 在 nfs 挂载的 nginx 目录里创建 ```index.html```
