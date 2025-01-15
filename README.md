# 智算中心监控部署手册
监控系统所依赖组件比较多，配置繁杂，所以请按照手册逐步操作。

首先先进行目录说明
- collector: 收集算网汇总指标所涉及到的组件。包括 snmp-exporter 与 opentelemetry-collector 两个组件的镜像及其配置。
- engine: 包含 prometheus, loki 两个收集引擎。
- exporter: 采集端需要的所有内容。包括打包好的 3个exporter 镜像和 categraf及其配置。
- k8s: 涉及到 k8s 的相关监控与采集。主要包含 prometheus-operator 与 k8s 相关指标。
- platform: 平台系统所需内容。包含mysql、redis、应用程序 3类组件，以及相关配置文件。
- xtrace: 监控平台系统的后端源码说明，以及如何构建镜像。


# 安装环境
- GPU 服务器、 Linux 操作系统
- 请提前安装好 Docker、docker-compose

# 操作步骤
## 第一步: 安装 k8s 相关内容。
> 如果没有使用 k8s，可以跳过该步骤

主要安装 k8s集群、prometheus-operator 以及相关指标采集器。

具体请参考 [k8s 环境的安装手册](./k8s/README-zh.md)

## 第二步: 安装各类exporters。 
主要有三个exporter。该步骤暴露的节点端口有:
```shell
node-exporter: 9100
dcgm-exporter: 9400
nvidia-smi-exporter: 9835
```

详情请参考: [exporters安装手册](./exporter/README-zh.md)


## 第三步: 安装汇总采集器。
包含 snmp-exporter、otel-collector 两个组件。该步骤暴露的节点端口有:
```shell
snmp-exporter: 9116
otel-collector: 8889
```

详情请参考: [汇总采集器](./collector/README-zh.md)


## 第四步: 安装引擎。 
即以容器的方式运行 prometheus 与 loki
> 如果已经使用 k8s，并且没有采集日志的需求，那这一步可以完全跳过

详情请参考: [引擎安装手册](./engine/README-zh.md)


## 第五步: 安装平台系统。
即我们的应用系统的安装，涉及 ibn、监控前端、监控后端三个服务；mysql, redis 两个中间件。

涉及到端口有:
```shell
mysql: 3306
redis: 6379
后端服务接口: 17000
前端服务接口: 8081
```

详情请参考: [平台系统安装手册](./plateform/README-zh.md)

# 镜像问题
如果由于网络问题无法拉取镜像，我们将需要的镜像(x86)全部分批打包好。

# 注意事项
当我们的 xtrace-backend 和 xtrace-ui 代码更新后，需要重新构建其镜像。然后注意修改 ./plateform/docker-compose.yaml 中的镜像数据:
```yaml
  n9e:
    container_name: n9e
    image: xtrace-backend:v1.0.0 # 注意修改镜像版本
  tracex-ui:
    container_name: tracex-ui
    image: xtrace-n9e-ui:test-202518-1751 # 注意修改镜像版本
    ports:
      - 8081:2015
    environment:
      - API_HOST=后端地址 # 修改出
      - IBN_HOST=ibn地址  # 修改出
    restart: always
    depends_on:
      - n9e
```


