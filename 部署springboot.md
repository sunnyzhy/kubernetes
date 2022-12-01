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

#### ```server-block.conf```

```conf
upstream lb_minio {
	server minio-cluster-0.minio-cluster-headless.iot:9000;
	server minio-cluster-1.minio-cluster-headless.iot:9000;
	server minio-cluster-2.minio-cluster-headless.iot:9000;
	server minio-cluster-3.minio-cluster-headless.iot:9000;
}

upstream lb_minio_ui {
	server minio-cluster-0.minio-cluster-headless.iot:9001;
	server minio-cluster-1.minio-cluster-headless.iot:9001;
	server minio-cluster-2.minio-cluster-headless.iot:9001;
	server minio-cluster-3.minio-cluster-headless.iot:9001;
}

server {
  listen 0.0.0.0:9000;

  location / {
	proxy_pass http://lb_minio;
  }
}

server {
  listen 0.0.0.0:9001;

  location / {
	proxy_pass http://lb_minio_ui;
  }
}
```

***注:***

- ```nginx``` 集群的 ```ConfigMap``` 名称是 ```nginx-cluster-server-block```
- ```upstream``` 格式为 ```server <pod_name>.<headless_service_name>.<namespace>:9000;```

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
    value: "http://192.168.5.100:30090/"
```

注: ```MINIO_SERVER``` 的格式为 ```http://<vip>:<nodePort>/```
