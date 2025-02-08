# 节点日志采集
这个模块主要针对从 k8s 集群采集，或则直接从节点上采集日志。

我们主要使用 promtail 来采集日志，它与 loki 衔接得非常丝滑。如果我们没有部署 loki，那么这部分内容可以跳过。

## k8s 中的日志采集
进入目录后，我们只需修改 `promtail-daemonset.yaml` 中一处信息:
```yaml
    clients:
      - url: http://loki节点ip:3100/loki/api/v1/push #修改处 loki地址
```
修改完 loki 地址后，直接执行执行:
```shell
% kubectl apply -f ./promtail-daemonset.yaml
```

随后我们执行如下命令，会看到两个相关pod，如果它们都是 Running 状态则代表部署成功:
```shell
$ k get pods -A | grep promtail
default  promtail-daemonset-4ng2m  1/1  Running 0 28h
default  promtail-daemonset-n8nnh  1/1 Running 0 28h
```

这时如果我们安装了 grafana，进入 grafana 配置好 loki 数据源后进入 Explore 界面，即可看到相关的日志指标。

## 节点上的日志采集
我们在没有使用 k8s 集群的情况，我们直接从每个节点上采集日志。

### 修改配置
启动前，请修改`./config/promtail-config.yaml`中的相关内容(`#修改处`)。

### 启动 promtail 容器采集日志
修改完配置后，直接运行下面 docker 命令即可。(记得修改 your_log_path)
```shell
docker run --name promtail -v ./config:/mnt/config -v ./your_log_path:/var/log grafana/promtail:3.2.1 --config.file=/mnt/config/promtail-config.yaml
```
