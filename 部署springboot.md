# 部署 springboot

## 前言

- 准备三台物理机
    |物理机IP|物理机HostName|角色|VIP|
    |--|--|--|--|
    |192.168.5.163|centos-docker-163|manager|192.168.5.100|
    |192.168.5.164|centos-docker-164|worker|192.168.5.100|
    |192.168.5.165|centos-docker-165|worker|192.168.5.100|

## eureka server

### ```application.yml```

```yml
eureka:
    client:
        fetchRegistry: true
        registerWithEureka: true
        serviceUrl:
            defaultZone: ${EUREKA_SERVER}
    instance:
        hostname: ${EUREKA_INSTANCE_HOSTNAME}
        perferIpAddress: true
    server:
        enable-self-preservation: false
        eviction-interval-timer-in-ms: 5000
server:
    port: 8080
spring:
    application:
        name: eureka-server
    security:
        user:
            name: eureka
            password: eureka
```

### ```StatefulSet.yml```

```yml
apiVersion: v1
kind: Service
metadata:
  name: eureka-headless
  namespace: iot
  labels:
    app: eureka-headless
spec:
  clusterIP: None
  ports:
    - port: 8080
      targetPort: 8080
      name: eureka-service
  selector:
    app: eureka-server
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka-server
  namespace: iot
spec:
  serviceName: eureka-headless
  replicas: 2
  selector:
    matchLabels:
      app: eureka-server
  template:
    metadata:
      labels:
        app: eureka-server
    spec:
      containers:
        - name: eureka-server
          image: core.harbor.domain/iot/eureka-server
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          resources:
            limits:
              memory: 512Mi
          env:
            - name: EUREKA_INSTANCE_HOST_NAME
              value: ${HOSTNAME}.eureka-headless
            - name: EUREKA_SERVER
              value: "http://eureka:eureka@eureka-server-0.eureka-headless.iot:8080/eureka,http://eureka:eureka@eureka-server-1.eureka-headless.iot:8080/eureka"
---

apiVersion: v1
kind: Service
metadata:
  name: eureka-server
  namespace: iot
  labels:
    app: eureka-server
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
  selector:
    app: eureka-server
```

注: ```EUREKA_SERVER``` 的格式为 ```http://<spring.security.user.name>:<spring.security.user.password>@<pod_name>.<headless_service_name>.<namespace>:8080/eureka```

## eureka client

### 配置 eureka server

#### ```application.yml```

```yml
eureka:
    client:
        fetch-registry: true
        serviceUrl:
            defaultZone: ${EUREKA_SERVER}
    instance:
        prefer-ip-address: true
```

#### ```StatefulSet.yml```

在 spec.template.spec.containers 里配置:

```yml
env:
  - name: EUREKA_SERVER
    value: "http://eureka:eureka@eureka-server-0.eureka-headless.iot:8080/eureka,http://eureka:eureka@eureka-server-1.eureka-headless.iot:8080/eureka"
```

注: ```EUREKA_SERVER``` 的格式为 ```http://<spring.security.user.name>:<spring.security.user.password>@<pod_name>.<headless_service_name>.<namespace>:8080/eureka```

### 配置 redis

#### ```application.yml```

```yml
spring:
    redis:
        database: 0
        lettuce:
            pool:
                max-active: 8
                max-idle: 8
                max-wait: -1
                min-idle: 1
        password: password
        timeout: 3000
        cluster:
            nodes: ${REDIS_SERVER}
```

#### ```StatefulSet.yml```

在 spec.template.spec.containers 里配置:

```yml
env:
  - name: REDIS_SERVER
    value: "redis-cluster-0.redis-cluster-headless.iot:6379,redis-cluster-1.redis-cluster-headless.iot:6379,redis-cluster-2.redis-cluster-headless.iot:6379,redis-cluster-3.redis-cluster-headless.iot:6379,redis-cluster-4.redis-cluster-headless.iot:6379,redis-cluster-5.redis-cluster-headless.iot:6379"
```

注: ```REDIS_SERVER``` 的格式为 ```<pod_name>.<headless_service_name>.<namespace>:6379```，多个值之间用逗号 ```,``` 隔开。

### 配置 myql

#### ```application.yml```

```yml
spring:
    datasource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        druid:
            filter:
                stat:
                    enabled: false
            initial-size: 1
            max-active: 20
            min-idle: 1
            stat-view-servlet:
                allow: true
            test-on-borrow: true
        password: root
        type: com.alibaba.druid.pool.DruidDataSource
        url: jdbc:mysql://${MYSQL_SERVER}:3306/db_name?characterEncodeing=utf-8&useSSL=false&serverTimezone=Asia/Shanghai
        username: root
```

#### ```StatefulSet.yml```

在 spec.template.spec.containers 里配置:

```yml
env:
  - name: MYSQL_SERVER
    value: "mysql-cluster-primary"
```

注: ```MYSQL_SERVER``` 的格式为 ```<service_name>```

### 配置 minio

#### 架构

|192.168.5.163|192.168.5.164|192.168.5.165|
|--|--|--|
||pod: minio-cluster-0, minio-cluster-1|pod: minio-cluster-2, minio-cluster-3|
||pod: jar-0|pod: jar-1|
|物理机: nginx|物理机: nginx|物理机: nginx|
|物理机: VIP|物理机: VIP|物理机: VIP|

#### minio-service.yml

```
spec:
  ports:
    - name: minio-api
      protocol: TCP
      port: 9000
      targetPort: minio-api
      nodePort: 30900
    - name: minio-console
      protocol: TCP
      port: 9001
      targetPort: minio-console
      nodePort: 30901
```

#### ```nginx.conf```

```conf
   upstream lb_minio {
        server 192.168.5.164:30900;
        server 192.168.5.165:30900;
   }

   upstream lb_minio_ui {
        server 192.168.5.164:30901;
        server 192.168.5.165:30901;
   }

    server {
        listen       9000;
        server_name  localhost;

        location / {
           proxy_set_header Host $http_host;
           proxy_pass http://lb_minio;
        }
    }

    server {
        listen       9001;
        server_name  localhost;

        location / {
           proxy_set_header Host $http_host;
           proxy_pass http://lb_minio_ui;
        }
    }

    server {
        listen       80;
        server_name  localhost;

        location /upload/ {
            proxy_set_header Host $http_host;
            proxy_pass http://lb_minio/;
        }
    }
```

***注:***

- ```upstream``` 格式为: ***```server <物理机IP>:<nodePort>;```***,比如: ```192.168.5.164:30900```, ***```server``` 的个数与 ```worker``` 节点的个数一致***
- 外部访问 ```minio-api``` 端口时映射到 pod 的流程: ```<VIP>:9000``` -> ```<物理机IP>:30900``` -> ```<Cluster IP>:9000``` -> ```<Endpoints>```
- 外部访问 ```minio-console``` 端口时映射到 pod 的流程: ```<VIP>:9001``` -> ```<物理机IP>:30901``` -> ```<Cluster IP>:9001``` -> ```<Endpoints>```
- 外部访问 ```minio api```: ***```http://<VIP>:9000```***，比如: ```http://192.168.5.100:9000```
- 外部访问 ```minio console```: ***```http://<VIP>:9001```***，比如: ```http://192.168.5.100:9001```
- 外部访问 ```minio 资源```: ***```http://<VIP>/upload/<resource>```***，比如: ```http://192.168.5.100/upload/xx.jpg```

#### ```application.yml```

```yml
minio:
    endpoint: ${MINIO_SERVER}
    access-key: admin
    secret-key: password
```

#### ```StatefulSet.yml```

在 spec.template.spec.containers 里配置:

```yml
env:
  - name: MINIO_SERVER
    value: "http://192.168.5.100:9000"
```

注: ```MINIO_SERVER``` 的格式为 ```http://<VIP>:9000```
