# 算网中心监控部署手册
监控系统所依赖组件比较多，配置繁杂，所以请按照手册逐步操作。

首先先进行目录说明
- collector: 收集算网汇总指标所涉及到的组件。包括 snmp-exporter 与 opentelemetry-collector 两个组件的镜像及其配置。
- engine: 包含 prometheus, loki 两个收集引擎。
- exporter: 采集端需要的所有内容。包括打包好的 3个exporter 镜像和 categraf及其配置。
- k8s: 涉及到 k8s 的相关采集。
- platform: 平台系统所需内容。包含mysql、redis、应用程序 3类组件，以及相关配置文件。
- xtrace: 监控平台系统的后端源码说明，以及如何构建镜像。


# 安装环境
- GPU 服务器、 Linux 操作系统
- 请提前安装好 Docker、docker-compose

# 操作步骤

第1步: 安装各类exporters。 [exporters安装手册](./exporter/README-zh.md)

第2步: 安装 k8s 相关采集。[k8s相关采集](./k8s/README-zh.md)

> 如果没有使用 k8s，可以跳过该步骤

第3步: 安装汇总采集器。[汇总采集器](./collector/README-zh.md)

第4步: 安装引擎。 [引擎安装手册](./engine/README-zh.md)

第5步: 安装平台系统。[平台系统安装手册](./plateform/README-zh.md)

# 镜像问题
如果由于网络问题无法拉取镜像，我们将需要的镜像(x86)全部分批打包好。可以到这下载[镜像链接](https://pan.baidu.com/s/1mkppNyGjJKvMgbOVvqI07w?pwd=de5q)

# 注意事项
当我们的 xtrace-backend 代码更新后，需要重新构建其镜像。然后注意修改 ./plateform/docker-compose.yaml 中的镜像数据:
```yaml
  n9e:
    container_name: n9e
    image: xtrace-backend:v1.0.0 # 修改
```


