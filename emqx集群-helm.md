# emqx 集群 - helm

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|
    |--|--|--|
    |192.168.5.163|centos-docker-163|manager|
    |192.168.5.164|centos-docker-164|worker|
    |192.168.5.165|centos-docker-165|worker|

- nfs 根目录: ```/nfs/data```

- 容器的 certs 目录:

    ```
    /opt/emqx/etc/certs
    ├── cacert.pem
    ├── cert.pem
    ├── client-cert.pem
    ├── client-key.pem
    ├── key.pem
    └── README
    ```

## 创建 emqx 目录

```bash
# mkdir -p /usr/local/k8s/emqx
```

## 生成证书

参考 [生成证书](./生成证书.md '生成证书')

***注: 证书里的域名配置的是 ```emqx.svc.cluster```:***

```
[alt_names]
DNS.1 = emqx.svc.cluster
```

## 添加 chart

```bash
# helm repo add emqx https://repos.emqx.io/charts

# helm repo update
```

## 查找 chart

```bash
# helm search repo emqx
NAME              	CHART VERSION	APP VERSION	DESCRIPTION                              
emqx/emqx         	5.0.3        	5.0.3      	A Helm chart for EMQX                    
emqx/emqx-ee      	4.4.6        	4.4.6      	A Helm chart for EMQ X                   
emqx/emqx-operator	1.0.10       	1.2.5      	A Helm chart for EMQX Operator Controller
emqx/kuiper       	0.9.0        	0.9.0      	A lightweight IoT edge analytic software 
```

## 拉取 chart

```bash
# helm pull emqx/emqx -d /usr/local/k8s/emqx
```

## 修改 chart 配置

```bash
# tar zxvf /usr/local/k8s/emqx/emqx-5.0.3.tgz -C /usr/local/k8s/emqx
```

### 修改 values.yaml

```bash
# vim /usr/local/k8s/emqx/emqx/values.yaml
```

```yml
emqxConfig:
  EMQX_DASHBOARD__DEFAULT_PASSWORD: root
  EMQX_DASHBOARD__DEFAULT_USERNAME: admin

persistence:
  enabled: true
  storageClassName: "nfs-client"

tolerations: 
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"

cacertPem: |+
  -----BEGIN CERTIFICATE-----
  MIIDlTCCAn2gAwIBAgIJAKPVIR/zTEJVMA0GCSqGSIb3DQEBCwUAMGExCzAJBgNV
  BAYTAmNuMQswCQYDVQQIDAJjbjELMAkGA1UEBwwCY24xCzAJBgNVBAoMAmNuMQsw
  CQYDVQQLDAJjbjELMAkGA1UEAwwCY24xETAPBgkqhkiG9w0BCQEWAmNuMB4XDTIy
  MDgxMjAyNTIxMloXDTMyMDgwOTAyNTIxM1owYTELMAkGA1UEBhMCY24xCzAJBgNV
  BAgMAmNuMQswCQYDVQQHDAJjbjELMAkGA1UECgwCY24xCzAJBgNVBAsMAmNuMQsw
  CQYDVQQDDAJjbjERMA8GCSqGSIb3DQEJARYCY24wggEiMA0GCSqGSIb3DQEBAQUA
  A4IBDwAwggEKAoIBAQC1GlC7RbDGFle3dibEdZLQ4XT5UcEagUfUG/XLOx+lgJdE
  1eD77wVjjCGTarFdXwLBk8Oowr6ZMXMge8eQKURtvBNUf5bq8weEwkD45c9dKDk1
  b078iwgLXDH9e+AVjbJcJUFAlxB/4CGhBoZgT5Bf/o9+0XVhkBRxHhrCJBxwitm2
  RGWm/B2CVYzfeV/nEco37oS3O+Bjj5hHh+HkHcmRE2+aTPGb/xt842dvI8iKsuOI
  VTVeqHPFFUGPjplVr0hvCoIBhfTQYPjBvbWcAY1/nhBN5PPQ98gdxdDPpJ38ZkAp
  V2Hk2c03UFxcW1NatyIrmQzA24ppK4CyxFgbYa8PAgMBAAGjUDBOMB0GA1UdDgQW
  BBS8RqeQLNO1hWdDNJ5zz5OASPFxKjAfBgNVHSMEGDAWgBS8RqeQLNO1hWdDNJ5z
  z5OASPFxKjAMBgNVHRMEBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQCa5jClTi4H
  gINxV/eXQ59pCnyPyC80+rgrf0YX7lN6UIhQ0ZowMtsAaj+rriAuGVOO8e9nkCdL
  WwX/PHGUZzKC5jvE+kpe6V/frqOl536hYyNgnVrSpV24ZP5qGXepRMRYHO/VIN2Z
  risCl4FWkuCq8I0j4EQZoWR/C8vWVgN3NLkmEobCi7arl3k1FkIHqi7FnbgESti3
  X10xwlfZdsTtANdYLizc1cImmMHUfpFhiKLjcvSwCInjxnawI3Zl+boiHUFdudWy
  N0GyvdQLoybX6wGFqDB2O7UIu4HUdYO2F871EsrNo+Ix/JF3yWQTyqXjanm/Smms
  SFRVBzBXvNZb
  -----END CERTIFICATE-----

keyPem: |+
  -----BEGIN RSA PRIVATE KEY-----
  MIIEowIBAAKCAQEAx9hLmTr9nSOE3K3IeDbMcrSWqGbDH9PWhVQeREtGzh6Oruhp
  o8nEKTufioe1juGcEI+Kz/hzYRbCG4gTyKmHiTnXRrcEc7yecUuuzob+vheuo4cQ
  3wZxeCaRCi5S2jkRz/13q0o5q+Qzlo45Neqlkh38NE1/NMX4T4EaaG0UlfWShlH3
  bsElTQnZO9P5LnCjo2EbrwLmV/bcaWMPSFmSziZvXNvLX0qG1M1QLrmWilWb5/bh
  HQDOFkHmjEIpU9T4F3H+i6oTK1edKJWSWIXTBRfgR3Jn6QyCAAZYvTMKtFd2wTnp
  42zBRtW5W0gPKX3e5Vv2ti7A0bW1HN3+/gB0sQIDAQABAoIBAHOV3Xu9y8vkLEc6
  2DT3dN5vJqg172tsKaop0eusIkK7F8mAMQGDhEpY6tjDzzXZtVs6uo245HrmH9Su
  qsqnivXKl4GV/yQ1lupuXKelGpe1uNNqHk7wOhbFL6RU0KTY1z2WG23d22dY8KGG
  lv5hPew6HZHetNFAI3NQd4aoeHyWai4DBwDapKNdxnwv8KhK6+8aoruEriychc+1
  9+l0ItCihgKtRNfgCKnWSfkGZxpyFcQ9h3kVfyaziZsrggUDl2hLWKDv85hH/DcB
  Kfm82G33zHjvgAgO5u6IXa353NwzisIjZ2rWbZofAA7MJMblyZpPrc7d9ypu0jfQ
  Vph9UEECgYEA/KkdqcNZUpPXQJ4D/s6F4mDJjuGfMABEk0+Cf/tHgcIV9hSP4U/7
  gIr+RBAr/p3apoPzdebcwEvIPTbPPfjwI8gOtQuM7x46rq3YhE6aMqTpTuhD9+FB
  POmsd/ajomc1w/XX+K7Qyo8jN93FuQJvli6AHCSpD0Q/x871y5H5LbkCgYEAynx5
  3d6BszlXDcRQAmxtgWq+7OLzjGgd5YXGtYR9x6tgLwKkuqJYSzYB1i6Vex7A91xO
  K8UEIJ504LPrVIwtrc7RndNXxC41iDN6NOMInnxlvioQQx8Dh0qI/ZdqXwBAUfEB
  qgGPEWJjDAme3f1vfmmZ3MDCY5rqBqBi9C5murkCgYA+4AL+vRtcXmghr3EFFvXg
  aP+fq+wGcPqfwhiSqsCXvOnkLCfGBTZsPaqa239nRHcmGw4aGBJ1VCaHfT1DidCQ
  xvdJ/I7c7qPh1mdcPYdH9RwOQ+SuKgMnGmuOCvWbs52An4xYWYCh987m1yyeJsoI
  EJhW6JSdtPk+SvWF7FSbGQKBgDVa6W3aNMj6IgNBqGeLFKvWaerXvp9PzBjv0DWC
  bPZRJS4d/zQilSt0VrWOXtsrdmN30i1cY8MInfmbzzXfHiY/aiKxTt4l4vs5vUj0
  ThAc7V5XjSNyp6ZdsI1J+cCAu3IcFHksoftNtpW0b0VFg7yGrQFrVY2Tg3ktiAWZ
  uH4ZAoGBAOmEsksn7tuJq3BRYHofDibTWJ/C0jIvq4Oog+O5EzB4QJSgsasxCQmC
  1VLYp2KvSMtRdcSDLdO+jsoGSVj3cXKE3uB/RTlM/NEOugXu1JaFkW8aTv6ELLG5
  nw/f9kunBAIFgd3Uc5mH9EVAsCIqrRTNPc0R1osmu75P/P3QsxTd
  -----END RSA PRIVATE KEY-----

certPem: |+
  -----BEGIN CERTIFICATE-----
  MIIDTjCCAjagAwIBAgIJANzPxfHoG5n6MA0GCSqGSIb3DQEBCwUAMGExCzAJBgNV
  BAYTAmNuMQswCQYDVQQIDAJjbjELMAkGA1UEBwwCY24xCzAJBgNVBAoMAmNuMQsw
  CQYDVQQLDAJjbjELMAkGA1UEAwwCY24xETAPBgkqhkiG9w0BCQEWAmNuMB4XDTIy
  MDgxMjAyNTkxN1oXDTMyMDgwOTAyNTkxN1owSzELMAkGA1UEBhMCQ04xDjAMBgNV
  BAgMBUh1YmVpMQ4wDAYDVQQHDAVXdWhhbjEPMA0GA1UECgwGU0VSVkVSMQswCQYD
  VQQDDAJDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMfYS5k6/Z0j
  hNytyHg2zHK0lqhmwx/T1oVUHkRLRs4ejq7oaaPJxCk7n4qHtY7hnBCPis/4c2EW
  whuIE8iph4k510a3BHO8nnFLrs6G/r4XrqOHEN8GcXgmkQouUto5Ec/9d6tKOavk
  M5aOOTXqpZId/DRNfzTF+E+BGmhtFJX1koZR927BJU0J2TvT+S5wo6NhG68C5lf2
  3GljD0hZks4mb1zby19KhtTNUC65lopVm+f24R0AzhZB5oxCKVPU+Bdx/ouqEytX
  nSiVkliF0wUX4EdyZ+kMggAGWL0zCrRXdsE56eNswUbVuVtIDyl93uVb9rYuwNG1
  tRzd/v4AdLECAwEAAaMfMB0wGwYDVR0RBBQwEoIQZW1xeC5zdmMuY2x1c3RlcjAN
  BgkqhkiG9w0BAQsFAAOCAQEAHWc3roVk4FyUaO1t7lGe2qY4Kf5mof/HCJvFl8MA
  5O5RJu9KaHzh3kB4VworLN437Akn4g8bat4ZrdHioCJOiZRW/lFuPw2eLPLX225/
  XkKF2RMZ3T+PEFOY5TvRHtUKKj4aoSK6lFuVynMFNNW1n8taqzOCQbUq28taIOTf
  YeY+q6TpFNAblKyNVeJkRinM9N7s/7rGKcyyi/lqvxC/GQHdF7fTC+XM4q0eyvzW
  udGwnNkAbsP6zRB07JhAXLio/g3zUq3Q2jz8MMk02fo/jSjuK2zAD2eqtKuTKF4e
  tDoGmgzbuI0tuhI0CrbP5P0Su4m92AO2G95ahZcQKGTeWw==
  -----END CERTIFICATE-----
```

### 修改 configmap.yaml

```bash
# vim /usr/local/k8s/emqx/emqx/templates/configmap.yaml
```

```yml
{{- if .Values.emqxConfig }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "emqx.fullname" . }}-env
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ include "emqx.name" . }}
    helm.sh/chart: {{ include "emqx.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
data:
  {{- range $index, $value := .Values.emqxConfig }}
  {{- if $value }}
  {{- $key := (regexReplaceAllLiteral "\\." (regexReplaceAllLiteral "EMQX[_\\.]" (upper (trimAll " " $index)) "") "__") }}
  {{ print "EMQX_" $key }}: "{{ tpl (printf "%v" $value) $ }}"
  {{- end }}
  {{- end }}
{{- end }}

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "emqx.fullname" . }}-ssl-cacert
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ include "emqx.name" . }}
    helm.sh/chart: {{ include "emqx.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
data:
  "cacert.pem": |+
    {{ .Values.cacertPem | nindent 4 }}

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "emqx.fullname" . }}-ssl-key
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ include "emqx.name" . }}
    helm.sh/chart: {{ include "emqx.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
data:
  "key.pem": |+
    {{ .Values.keyPem | nindent 4 }}

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "emqx.fullname" . }}-ssl-cert
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ include "emqx.name" . }}
    helm.sh/chart: {{ include "emqx.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
data:
  "cert.pem": |+
    {{ .Values.certPem | nindent 4 }}
```

### 修改 StatefulSet.yaml

```bash
# vim /usr/local/k8s/emqx/emqx/templates/StatefulSet.yaml
```

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "emqx.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ include "emqx.name" . }}
    helm.sh/chart: {{ include "emqx.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  serviceName: {{ include "emqx.fullname" . }}-headless
  podManagementPolicy: {{ .Values.podManagementPolicy }}
  {{- if and .Values.persistence.enabled (not .Values.persistence.existingClaim) }}
  volumeClaimTemplates:
    - metadata:
        name: emqx-data
        namespace: {{ .Release.Namespace }}
        labels:
          app.kubernetes.io/name: {{ include "emqx.name" . }}
          app.kubernetes.io/instance: {{ .Release.Name }}
          app.kubernetes.io/managed-by: {{ .Release.Service }}
      spec:
        {{- if .Values.persistence.storageClassName }}
        storageClassName: {{ .Values.persistence.storageClassName | quote }}
        {{- end }}
        accessModes:
          - {{ .Values.persistence.accessMode | quote }}
        resources:
         requests:
           storage: {{ .Values.persistence.size | quote }}
  {{- end }}
  updateStrategy:
    type: RollingUpdate
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "emqx.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ include "emqx.name" . }}
        version: {{ .Chart.AppVersion }}
        app.kubernetes.io/name: {{ include "emqx.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
      annotations:
      {{- with .Values.podAnnotations }}
          {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- if .Values.recreatePods }}
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum | quote }}
      {{- end }}
    spec:
      volumes:
      {{- if not .Values.persistence.enabled }}
      - name: emqx-data
        emptyDir: {}
      {{- else if .Values.persistence.existingClaim }}
      - name: emqx-data
        persistentVolumeClaim:
        {{- with .Values.persistence.existingClaim }}
          claimName: {{ tpl . $ }}
        {{- end }}
      {{- end }}
      {{- if .Values.emqxLicenseSecretName  }}
      - name: emqx-license
        secret:
          secretName: {{ .Values.emqxLicenseSecretName }}
      {{- end }}
      - name: emqx-timezone
        hostPath:
          path: /etc/timezone
      - name: emqx-ssl-cacert
        configMap:
          name: {{ include "emqx.fullname" . }}-ssl-cacert
          items:
          - key: cacert.pem
            path: cacert.pem
      - name: emqx-ssl-key
        configMap:
          name: {{ include "emqx.fullname" . }}-ssl-key
          items:
          - key: key.pem
            path: key.pem
      - name: emqx-ssl-cert
        configMap:
          name: {{ include "emqx.fullname" . }}-ssl-cert
          items:
          - key: cert.pem
            path: cert.pem
      {{- if eq .Values.emqxConfig.EMQX_CLUSTER__DISCOVERY_STRATEGY "k8s"}}
      serviceAccountName:  {{ include "emqx.fullname" . }}
      {{- end }}
      {{- if .Values.podSecurityContext.enabled }}
      securityContext: {{- omit .Values.podSecurityContext "enabled" | toYaml | nindent 8 }}
      {{- end }}
      {{- if .Values.initContainers }}
      initContainers:
{{ toYaml .Values.initContainers | indent 8 }}
      {{- end }}
      {{- if .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- range .Values.image.pullSecrets }}
        - name: {{ . }}
      {{- end }}
      {{- end }}
      containers:
        - name: emqx
          image: "{{ .Values.image.repository }}:{{ .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- if .Values.containerSecurityContext.enabled }}
          securityContext: {{- omit .Values.containerSecurityContext "enabled" | toYaml | nindent 12 }}
          {{- end }}
          ports:
          - name: mqtt
            containerPort: {{ .Values.emqxConfig.EMQX_LISTENERS__TCP__DEFAULT | default 1883 }}
          - name: mqttssl
            containerPort: {{ .Values.emqxConfig.EMQX_LISTENERS__SSL__DEFAULT | default 8883 }}
          - name: ws
            containerPort: {{ .Values.emqxConfig.EMQX_LISTENERS__WS__DEFAULT | default 8083 }}
          - name: wss
            containerPort: {{ .Values.emqxConfig.EMQX_LISTENERS__WSS__DEFAULT | default 8084 }}
          - name: dashboard
            containerPort: {{ .Values.emqxConfig.EMQX_DASHBOARD__LISTENER__HTTP | default 18083 }}
          {{- if not (empty .Values.emqxConfig.EMQX_LISTENERS__TCP__DEFAULT) }}
          - name: internalmqtt
            containerPort: {{ .Values.emqxConfig.EMQX_LISTENERS__TCP__DEFAULT }}
          {{- end }}
          {{- if not (empty .Values.emqxConfig.EMQX_DASHBOARD__LISTENER__HTTPS) }}
          - name: dashboardtls
            containerPort: {{ .Values.emqxConfig.EMQX_DASHBOARD__LISTENER__HTTPS }}
          {{- end }}
          - name: ekka
            containerPort: 4370
          envFrom:
            - configMapRef:
                name: {{ include "emqx.fullname" . }}-env
         {{- if .Values.envFromSecret }}
            - secretRef:
                name: {{ .Values.envFromSecret }}
         {{- end }}
          resources:
{{ toYaml .Values.resources | indent 12 }}
          volumeMounts:
          - name: emqx-data
            mountPath: "/opt/emqx/data"
          {{ if .Values.emqxLicenseSecretName  }}
          - name: emqx-license
            mountPath: "/opt/emqx/etc/emqx.lic"
            subPath: "emqx.lic"
            readOnly: true
          {{ end }}
          - name: emqx-timezone
            mountPath: /etc/timezone
          - name: emqx-ssl-cacert
            mountPath: /opt/emqx/etc/certs/cacert.pem
            subPath: cacert.pem
          - name: emqx-ssl-key
            mountPath: /opt/emqx/etc/certs/key.pem
            subPath: key.pem
          - name: emqx-ssl-cert
            mountPath: /opt/emqx/etc/certs/cert.pem
            subPath: cert.pem
          readinessProbe:
            httpGet:
              path: /api/v5/status
              port: {{ .Values.emqxConfig.EMQX_DASHBOARD__LISTENER__HTTP | default 18083 }}
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /api/v5/status
              port: {{ .Values.emqxConfig.EMQX_DASHBOARD__LISTENER__HTTP | default 18083 }}
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 10
    {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
    {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
    {{- end }}
    {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
    {{- end }}
```

## 重新制作 chart

```bash
# rm -rf /usr/local/k8s/emqx/emqx-5.0.3.tgz

# helm package /usr/local/k8s/emqx/emqx -d /usr/local/k8s/emqx

# tree /usr/local/k8s/emqx
/usr/local/k8s/emqx
├── emqx
│   ├── Chart.yaml
│   ├── README.md
│   ├── templates
│   │   ├── configmap.yaml
│   │   ├── _helpers.tpl
│   │   ├── ingress.yaml
│   │   ├── rbac.yaml
│   │   ├── secret.yaml
│   │   ├── service-monitor.yaml
│   │   ├── service.yaml
│   │   └── StatefulSet.yaml
│   └── values.yaml
└── emqx-5.0.3.tgz
```

## 部署

```bash
# helm install emqx-cluster /usr/local/k8s/emqx/emqx-5.0.3.tgz -n iot
NAME: emqx-cluster
LAST DEPLOYED: Thu Aug 11 15:43:47 2022
NAMESPACE: iot
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

查看资源:

```bash
# kubectl get sts -n iot | grep emqx
emqx-cluster                         3/3     115s

# kubectl get pod -n iot -o wide | grep emqx
emqx-cluster-0                                     1/1     Running            0                4m4s   10.244.1.112   centos-docker-164   <none>           <none>
emqx-cluster-1                                     1/1     Running            0                4m4s   10.244.2.89    centos-docker-165   <none>           <none>
emqx-cluster-2                                     1/1     Running            0                4m4s   10.244.1.111   centos-docker-164   <none>           <none>

# kubectl get pvc,pv -n iot | grep emqx
persistentvolumeclaim/emqx-data-emqx-cluster-0                Bound    pvc-96306b7a-a802-4ea7-8b89-afd7c839159d   20Mi       RWO            nfs-client     11m
persistentvolumeclaim/emqx-data-emqx-cluster-1                Bound    pvc-4019a423-b3f9-4740-af77-1eccccb0ad2c   20Mi       RWO            nfs-client     11m
persistentvolumeclaim/emqx-data-emqx-cluster-2                Bound    pvc-fd38d0c9-e7a5-4afe-9609-70afc7dc7e52   20Mi       RWO            nfs-client     11m
persistentvolume/pvc-4019a423-b3f9-4740-af77-1eccccb0ad2c   20Mi       RWO            Delete           Bound    iot/emqx-data-emqx-cluster-1                nfs-client              11m
persistentvolume/pvc-96306b7a-a802-4ea7-8b89-afd7c839159d   20Mi       RWO            Delete           Bound    iot/emqx-data-emqx-cluster-0                nfs-client              11m
persistentvolume/pvc-fd38d0c9-e7a5-4afe-9609-70afc7dc7e52   20Mi       RWO            Delete           Bound    iot/emqx-data-emqx-cluster-2                nfs-client              11m

# kubectl get svc -n iot | grep emqx
emqx-cluster                            ClusterIP   10.99.86.156     <none>        1883/TCP,8883/TCP,8083/TCP,8084/TCP,18083/TCP                                                4m48s
emqx-cluster-headless                   ClusterIP   None             <none>        1883/TCP,8883/TCP,8083/TCP,8084/TCP,18083/TCP,4370/TCP                                       4m48s
```

## 内部访问 emqx 集群

略

## 外部访问 emqx 集群

创建 NodePort 类型的 Service:

```bash
# vim /usr/local/k8s/emqx/service.yaml
```

```yml
apiVersion: v1
kind: Service
metadata:
  name: emqx-cluster-service
  namespace: iot
spec:
  selector:
    app.kubernetes.io/instance: emqx-cluster
    app.kubernetes.io/name: emqx
  ports:
    - name: mqtt
      protocol: TCP
      port: 1883
      targetPort: mqtt
      nodePort: 30183
    - name: mqttssl
      protocol: TCP
      port: 8883
      targetPort: mqttssl
      nodePort: 30883
    - name: ws
      protocol: TCP
      port: 8083
      targetPort: ws
      nodePort: 30083
    - name: wss
      protocol: TCP
      port: 8084
      targetPort: wss
      nodePort: 30084
    - name: dashboard
      protocol: TCP
      port: 18083
      targetPort: dashboard
      nodePort: 31083
    - name: ekka
      protocol: TCP
      port: 4370
      targetPort: ekka
      nodePort: 30370
  type: NodePort
```

```bash
# kubectl apply -f /usr/local/k8s/emqx/service.yaml

# kubectl get svc emqx-cluster-service -n iot
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                      AGE
emqx-cluster-service   NodePort   10.96.246.44   <none>        1883:30183/TCP,8883:30883/TCP,8083:30083/TCP,8084:30084/TCP,18083:31083/TCP,4370:30370/TCP   17s
```

### 外部服务器访问 emqx 的 dashboard

在浏览器的地址栏里输入:```http://192.168.5.163:31083/```; ```用户名/密码```: ```admin/root```

### 普通连接 emqx 集群

![emqx-01](./images/emqx/emqx-01.png)

### ssl 连接 emqx 集群

1. 在外部服务器的 hosts 里添加域名映射: ```192.168.5.163 emqx.svc.cluster```

2. 获取 cacert.pem，可以用以下两种方法中的任意一种:

   - 从 pod 里拷贝 certs 目录:

        ```bash
        # kubectl exec -n iot emqx-cluster-0 -- tar cf - /opt/emqx/etc/certs | tar xf - -C /usr/local/k8s/emqx/
        ```

        把 ```/usr/local/k8s/emqx/opt/emqx/etc/certs/cacert.pem``` 拷贝到外部服务器的任意位置，比如: ```D:\certs\cacert.pem```
   
   - 直接把 ```/usr/local/k8s/emqx/emqx/values.yaml``` 里的 cacert.pem 内容拷贝到 ```D:\certs\cacert.pem```:

        ```pem
        -----BEGIN CERTIFICATE-----
        MIIDlTCCAn2gAwIBAgIJAKPVIR/zTEJVMA0GCSqGSIb3DQEBCwUAMGExCzAJBgNV
        BAYTAmNuMQswCQYDVQQIDAJjbjELMAkGA1UEBwwCY24xCzAJBgNVBAoMAmNuMQsw
        CQYDVQQLDAJjbjELMAkGA1UEAwwCY24xETAPBgkqhkiG9w0BCQEWAmNuMB4XDTIy
        MDgxMjAyNTIxMloXDTMyMDgwOTAyNTIxM1owYTELMAkGA1UEBhMCY24xCzAJBgNV
        BAgMAmNuMQswCQYDVQQHDAJjbjELMAkGA1UECgwCY24xCzAJBgNVBAsMAmNuMQsw
        CQYDVQQDDAJjbjERMA8GCSqGSIb3DQEJARYCY24wggEiMA0GCSqGSIb3DQEBAQUA
        A4IBDwAwggEKAoIBAQC1GlC7RbDGFle3dibEdZLQ4XT5UcEagUfUG/XLOx+lgJdE
        1eD77wVjjCGTarFdXwLBk8Oowr6ZMXMge8eQKURtvBNUf5bq8weEwkD45c9dKDk1
        b078iwgLXDH9e+AVjbJcJUFAlxB/4CGhBoZgT5Bf/o9+0XVhkBRxHhrCJBxwitm2
        RGWm/B2CVYzfeV/nEco37oS3O+Bjj5hHh+HkHcmRE2+aTPGb/xt842dvI8iKsuOI
        VTVeqHPFFUGPjplVr0hvCoIBhfTQYPjBvbWcAY1/nhBN5PPQ98gdxdDPpJ38ZkAp
        V2Hk2c03UFxcW1NatyIrmQzA24ppK4CyxFgbYa8PAgMBAAGjUDBOMB0GA1UdDgQW
        BBS8RqeQLNO1hWdDNJ5zz5OASPFxKjAfBgNVHSMEGDAWgBS8RqeQLNO1hWdDNJ5z
        z5OASPFxKjAMBgNVHRMEBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQCa5jClTi4H
        gINxV/eXQ59pCnyPyC80+rgrf0YX7lN6UIhQ0ZowMtsAaj+rriAuGVOO8e9nkCdL
        WwX/PHGUZzKC5jvE+kpe6V/frqOl536hYyNgnVrSpV24ZP5qGXepRMRYHO/VIN2Z
        risCl4FWkuCq8I0j4EQZoWR/C8vWVgN3NLkmEobCi7arl3k1FkIHqi7FnbgESti3
        X10xwlfZdsTtANdYLizc1cImmMHUfpFhiKLjcvSwCInjxnawI3Zl+boiHUFdudWy
        N0GyvdQLoybX6wGFqDB2O7UIu4HUdYO2F871EsrNo+Ix/JF3yWQTyqXjanm/Smms
        SFRVBzBXvNZb
        -----END CERTIFICATE-----
        ```

2. 连接 emqx 集群:

    ![emqx-02](./images/emqx/emqx-02.png)
