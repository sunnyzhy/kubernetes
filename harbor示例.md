# harbor 示例

## 镜像

### 准备事项

准备三台物理机:

|物理机IP|物理机HostName|角色|
|--|--|--|
|192.168.5.163|centos-docker-163|manager|
|192.168.5.164|centos-docker-164|worker|
|192.168.5.165|centos-docker-165|worker|

#### 创建 harbor 项目

在 harbor 上创建名称为 ```test``` 的项目。

#### 配置域名映射

```bash
# vim /etc/hosts
192.168.5.165 core.harbor.domain
```

#### 修改 docker 的启动参数

##### 修改参数

###### 方法一

```bash
# vim /usr/lib/systemd/system/docker.service
```

在 ```ExecStart``` 配置项的最末位置添加 ``` --insecure-registry core.harbor.domain```

###### 方法二

```bash
# vim /etc/docker/daemon.json
```

```json
{
  "registry-mirrors": ["https://6848w7y3.mirror.aliyuncs.com"],
  "insecure-registries": ["core.harbor.domain"]
}
```

在 ```daemon.json``` 里添加 ```"insecure-registries": ["core.harbor.domain"]```

###### 方法三

```bash
# mkdir -p /etc/docker/certs.d/core.harbor.domain

# cp ca.crt /etc/docker/certs.d/core.harbor.domain/ca.crt
```

把自签名的 ca 证书添加到信任。

##### 重启 docker

```bash
# systemctl daemon-reload

# systemctl restart docker
```

### 上传镜像

#### 创建 docker 镜像

创建 docker 镜像:

```bash
# docker build -t demo:1.0.0 .
```

给镜像打标签 ```docker tag <image name>:<image tag> core.harbor.domain/<harbor project name>/<image name>:<image tag>```:

```bash
# docker tag demo:1.0.0 core.harbor.domain/test/demo:1.0.0
```

#### 登录 harbor

##### 登录方式一

```bash
# docker login core.harbor.domain
Username: admin
Password: adminpassword
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

##### 登录方式二

```bash
# docker login -u admin -p adminpassword core.harbor.domain
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

#### 上传 docker 镜像

语法: ```docker push core.harbor.domain/<harbor project name>/<image name>:<image tag>```

```bash
# docker push core.harbor.domain/test/demo:1.0.0
The push refers to repository [core.harbor.domain/test/demo]
5a314e9b8af1: Pushed 
35c20f26d188: Pushed 
c3fe59dd9556: Pushed 
6ed1a81ba5b6: Pushed 
a3483ce177ce: Pushed 
ce6c8756685b: Pushed 
30339f20ced0: Pushed 
0eb22bfb707d: Pushed 
a2ae92ffcd29: Pushed 
2.12.1.6731: digest: sha256:655ddd6822d96e15d19757eb26d1896f11c108b76a539c76d344e305eaa01fda size: 2212
```

#### 查看上传的镜像

在 harbor 的 ```test``` 项目里查看 ```镜像仓库``` 选项页，会发现镜像列表里出现了 ```test/demo``` 镜像；点击镜像名称，可以查看镜像的详情。

## FAQ

```bash
# docker login core.harbor.domain
Username: admin
Password: adminpassword
Error response from daemon: Get "https://core.harbor.domain/v2/": x509: certificate signed by unknown authority
```

需要修改 docker 的启动参数，在 ```ExecStart``` 配置项的最末位置添加 ``` --insecure-registry core.harbor.domain```
