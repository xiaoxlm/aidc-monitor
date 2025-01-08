# 说明
这里监控 k8s 集群的模块。

这里会安装 kube-prometheus，来监控k8s集群相关内容。
主要监控 k8s 本身的资源信息和 pod 中的日志，用到了 kube-state-metrics 和 promtail 两个组件。
> 如果不需要收集日志，可以跳过 promtail 的安装


k8s 环境要部署的内容:
1. 先不用 dcgm-exporter
2. 重新安装

# 部署 kube-state-metrics
kube-state-metrics 主要用于暴露 k8s 的相关资源信息，比如Node、Deployment、Job、Pod 等。

首先我们把  [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 项目`git clone`到本地。然后执行如下命令部署:
```shell
kubectl apply -k examples/standard
```
这时我们执行如下命令:
```shell
$ kubectl get pods -A | grep kube-state-metrics

kube-system               kube-state-metrics-7b8966b979-h9whd                               1/1     Running             0              46h
```

看到 pod 是 Running 状态即部署成功了。

## 暴露指标
这时我们部署一个 kube-state-metrics 相关的 nodePort Service 用于暴露指标。

进入该目录后执行:
```shell
$ kubectl apply -f ./kube-state-metrics-nodeport.yaml
```

然后我们执行如下命令，如果有返回指标则说明指标暴露成功，可以采集了:
```shell
$ curl http://k8s节点ip:31666/metrics
```

# 部署 promtail
promtail 主要用日志的采集，它与 loki 衔接得非常丝滑。如果我们没有部署 loki，那么这部分内容可以跳过。

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