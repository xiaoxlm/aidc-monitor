# 说明
该模块涉及到k8s的相关部署，包括 k8s 集群的搭建、nvidia 相关的operator， prometheus-operator 及相关监控。
> 如果不需要收集日志，可以跳过 promtail 的安装

# 部署 k8s
请参考 [k8s部署手册](./build-cluster-manual-zh.md)

# 部署 prometheus-operator
部署 prometheus-operator 及其相关监控。包括了 kube-state-metrics 与 cadvisor 等指标。

首先进入到该目录 "./k8s" 后，按如下步骤执行:
1. 部署相关 crd
```shell
$ kubectl apply --server-side -f setup/
$ kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
```
2. 部署相关监控
```shell
$ kubectl apply -f kube-prometheus/
```

3. 部署 kube-state-metrics
```shell
kubectl apply -k kube-prometheus/kube-state-metrics/
```

然后我们看到 pod 已经 running 起来:
```shell
$ kubectl get pods -nmonitoring

NAME                                  READY   STATUS    RESTARTS        AGE
blackbox-exporter-5dfbb6c6b5-skvfz    3/3     Running   15 (60m ago)    6d23h
grafana-86bf6d9f44-7jvj6              1/1     Running   5 (60m ago)     6d23h
kube-state-metrics-7b8966b979-vmnpx   1/1     Running   43 (55m ago)    6d23h
prometheus-adapter-77f8587965-2lbf7   1/1     Running   6 (4h8m ago)    6d23h
prometheus-adapter-77f8587965-t9cs5   1/1     Running   9 (55m ago)     6d23h
prometheus-k8s-0                      2/2     Running   10 (60m ago)    6d23h
prometheus-k8s-1                      2/2     Running   12 (4h8m ago)   6d23h
prometheus-operator-5c868d654-grfqs   2/2     Running   12 (55m ago)    6d23h
```

因为我们已经部署了一个 NodePort
```shell
$ kubectl get svc prometheus-k8s-nodeport
NAME                      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
prometheus-k8s-nodeport   NodePort   10.105.149.238   <none>        9090:30900/TCP   6d23h
```

所以，我们在浏览器中输入 http://nodeIp:30900 就能看到 prometheus 的页面。同时进入 http://nodeIp:30900/targets 页面可以看到监控了哪些内容。

# 部署 dlrover
请参考 [dlrover部署手册](./dlrover-zh.md)