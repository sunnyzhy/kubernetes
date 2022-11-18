# containerd - ctr 和 crictl 的区别

```ctr``` 和 ```crictl``` 的区别:

1. ```ctr``` 是 ```containerd``` 自带的 CLI 命令行工具

2. ```crictl``` 是 ```kubernetes``` 中 CRI（容器运行时接口）的客户端，```kubernetes``` 使用 ```crictl``` 与 ```containerd``` 进行交互

3. ```ctr``` 的命名空间默认是 ```default```，查看所有命名空间:
   ```bash
   # ctr ns ls
   NAME    LABELS 
   default        
   k8s.io         
   moby           
   ```

4. ```crictl``` 有且仅有一个命名空间 ```k8s.io```

5. ```crictl image list``` 等价于 ```ctr -n k8s.io image list```
   - 指定命名空间拉取镜像
      ```bash
      ctr -n k8s.io image pull core.harbor.domain/<NAMESPACE>/<IMAGE_NAME>:<TAG>
      
      crictl image ls
      ```

***可以使用 ```crictl``` 来测试 ```kubernetes``` 能否成功拉取 harbor 仓库里的镜像。***
