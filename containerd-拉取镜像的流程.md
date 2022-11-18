# containerd - 拉取镜像的流程

## 前言

本示例以拉取 ```harbor``` 仓库里的镜像为例。

## 1. 解析镜像名

命令行:

```bash
# curl -v -u <HARBOR_USERNAME>:<HARBOR_PASSWORD> -X HEAD -H "Accept: application/vnd.docker.distribution.manifest.v2+json, application/vnd.docker.distribution.manifest.list.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, */*" https://<HARBOR_DNS>/v2/<HARBOR_PROJECT_NAME>/<IMAGE>/manifests/<TAG>
```

示例:

```bash
# curl -v -u admin:adminpassword -X HEAD -H "Accept: application/vnd.docker.distribution.manifest.v2+json, application/vnd.docker.distribution.manifest.list.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, */*" https://core.harbor.domain/v2/iot/busybox/manifests/latest
...
< HTTP/1.1 200 OK
< Server: nginx
< Date: Fri, 18 Nov 2022 02:16:07 GMT
< Content-Type: application/vnd.docker.distribution.manifest.v2+json
< Content-Length: 527
< Connection: keep-alive
< Docker-Content-Digest: sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee
< Docker-Distribution-Api-Version: registry/2.0
< Etag: "sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee"
< Set-Cookie: sid=4c051dda37c38eb80ecf965cb405ffa3; Path=/; HttpOnly
< X-Request-Id: 1457dd28-eec4-4bf3-b45e-4d22a9b1a487
< Strict-Transport-Security: max-age=31536000; includeSubdomains; preload
< X-Frame-Options: DENY
< Content-Security-Policy: frame-ancestors 'none'
```

响应的 ```Content-Type```:

- 如果是 ```application/vnd.docker.distribution.manifest.list.v2+json```，就执行步骤 ```2``` 可以获取镜像的 ```list``` 列表
- 如果是 ```application/vnd.docker.distribution.manifest.v2+json```，就跳过步骤 ```2```，直接执行步骤 ```3``` 以获取镜像的 ```manifest```

## 2. 获取镜像的 ```list``` 列表

命令行:

```bash
# curl -v -u <HARBOR_USERNAME>:<HARBOR_PASSWORD> -X HEAD -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json" https://<HARBOR_DNS>/v2/<HARBOR_PROJECT_NAME>/<IMAGE>/manifests/<Docker-Content-Digest>
```

示例:

```bash
# curl u admin:adminpassword -X GET -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json"  https://core.harbor.domain/v2/iot/busybox/manifests/sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1456,
      "digest": "sha256:beae173ccac6ad749f76713cf4440fe3d21d1043fe616dfbe30775815d1d0f6a"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 772788,
         "digest": "sha256:5cc84ad355aaa64f46ea9c7bbcc319a9d808ab15088a27209c9e70ef86e5a2aa"
      }
   ]
}
```

***注: 由于本示例响应的 ```Content-Type``` 是 ```application/vnd.docker.distribution.manifest.v2+json```，所以获取的镜像 ```list``` 列表就是镜像的 ```manifest```***

## 3. 获取镜像的 ```manifest```

命令行:

```bash
# curl -v -u <HARBOR_USERNAME>:<HARBOR_PASSWORD> -X HEAD -H "Accept: application/vnd.docker.distribution.manifest.v2+json" https://<HARBOR_DNS>/v2/<HARBOR_PROJECT_NAME>/<IMAGE>/manifests/<Docker-Content-Digest>
```

示例:

```bash
# curl -u admin:adminpassword -X GET -H "Accept: application/vnd.docker.distribution.manifest.v2+json"  https://core.harbor.domain/v2/iot/busybox/manifests/sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1456,
      "digest": "sha256:beae173ccac6ad749f76713cf4440fe3d21d1043fe616dfbe30775815d1d0f6a"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 772788,
         "digest": "sha256:5cc84ad355aaa64f46ea9c7bbcc319a9d808ab15088a27209c9e70ef86e5a2aa"
      }
   ]
}
```

## 4. 拉取镜像的 config 和 layers

解析步骤 ```3``` 里获取的 ```manifest```，分别下载镜像的 ```config``` 和 ```layers```

***拉取镜像的完整流程:***

```bash
# ctr -n k8s.io image pull core.harbor.domain/iot/busybox:latest
core.harbor.domain/iot/busybox:latest:                                            resolved       |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee: done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:5cc84ad355aaa64f46ea9c7bbcc319a9d808ab15088a27209c9e70ef86e5a2aa:    done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:beae173ccac6ad749f76713cf4440fe3d21d1043fe616dfbe30775815d1d0f6a:   done           |++++++++++++++++++++++++++++++++++++++| 
elapsed: 0.3 s                                                                    total:   0.0 B (0.0 B/s)                                         
unpacking linux/amd64 sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee...
done: 131.029994ms	
```
