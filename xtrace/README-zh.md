首先我们将 [xtrace-backend](https://github.com/xiaoxlm/xtrace-backend) 代码 clone 下来。

# 构建镜像
第一步我们进入工程，然后编译二进制文件:
```shell
make build-amd-linux
```

第二步构建镜像:
```shell
$ make build-image
```

最后我们会看到一个此镜像 `xtrace-backend:v1.0.0`
```shell
$ docker images | grep xtrace-backend

xtrace-backend   v1.0.0  be2736159534   7 seconds ago   116MB
```

