# 说明
这是各类指标采集模块。

我们需要在各个节点机器上采集所需的监控指标，
即我们需要到各个节点部署下文提到的 exporters 和 categraf

# 启动所有 exporter
进入到该目录后，我们直接使用 `docker-compose` 命令启动即可
```shell
$ docker-compose -f docker-compose.yaml up -d

✔ Container nvidia-smi-exporter  Started   0.6s
✔ Container node-exporter        Started   0.6s
✔ Container dcgm-exporter        Started   0.8s
```

> 启动的时候可能会遇到报错:`could not select device driver "" with capabilities: [[gpu]]`
> 
> 这是因为没有安装 nvidia-container-toolkit。按照如下步骤解决问题:
> 1. apt-get install -y nvidia-container-toolkit
> 2. systemctl restart docker


## 验证 exporter 启动成功
我们可以验证各个 exporter 是否采集成功。以下命令如果有 metrics 数据返回则代表采集成功:
```shell
# nvidia-smi-exporter
curl http://localhost:9835/metrics

# node-exporter
curl http://localhost:9100/metrics

# dcgm-exporter 启动会比较慢，需要等待30s左右
curl http://localhost:9400/metrics
```


# 启动 categraf
categraf 是一个代理程序，我们的节点上报以及执行告警治愈脚本，都依赖它。

首先我们先到这 [categraf-release](`https://github.com/flashcatcloud/categraf/releases`) 下载对应平台的软件包。下载完成后进入目录，编辑 `conf/config.toml ` 配置文件。核心修改内容如下:
```toml
[global]
#...
hostname = "..." # 节点唯一标识，可以填写节点 ip

[global.labels] # 标签信息，可以用于维度区分。key,value完全自定义
# ...
# key=value
device_type="GPU Server" # 设备类型标识
IBN="算网唯一标识" # 算网标识

[[writers]]
# ...
url = "http://后端系统ip:17000/prometheus/v1/write"  # 用于将采集的指标推送至prometheus。

[ibex]
enable = true 
## ibex flush interval
interval = "1000ms"
## n9e ibex server rpc address
servers = ["后端系统ip:20090"]
## temp script dir
meta_dir = "./meta"

[heartbeat] # 心跳检查
enable = true

url = "http://后端系统ip:17000/v1/n9e/heartbeat"
```
修改完配置后，使用如下命令启动:
```shell
nohup ./categraf &> categraf.log &
```

*但要注意!* 由于我们还没有启动后端系统，所有会看到探活失败的警告信息。 在后端系统启动后，警告信息会消失。 

建议在后端系统启动后，再来启动 categraf 也是可以的。

我们直接通过 `cat categraf.log` 查看日志内容来判断是否启动成功。如果没有看到报错信息，并且看到如下内容，代表启动成功:
```shell
2024/12/13 02:44:41 main.go:150: I! runner.hostname: node1
2024/12/13 02:44:41 main.go:151: I! runner.fd_limits: (soft=65535, hard=65535)
2024/12/13 02:44:41 main.go:152: I! runner.vm_limits: (soft=unlimited, hard=unlimited)
2024/12/13 02:44:41 provider_manager.go:60: I! use input provider: [local]
2024/12/13 02:44:41 prometheus_agent.go:19: I! prometheus scraping disabled!
2024/12/13 02:44:41 agent.go:38: I! agent starting
2024/12/13 02:44:41 metrics_agent.go:243: E! input: local.amd_rocm_smi not supported
2024/12/13 02:44:41 metrics_agent.go:243: E! input: local.arp_packet not supported
2024/12/13 02:44:41 bind.go:65: DEBUG: 0
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.conntrack started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.cpu started
2024/12/13 02:44:41 metrics_agent.go:243: E! input: local.dcgm not supported
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.disk started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.diskio started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.ethtool started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.greenplum started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.ipvs started
2024/12/13 02:44:41 metrics_agent.go:243: E! input: local.jolokia_agent_kafka not supported
2024/12/13 02:44:41 metrics_agent.go:243: E! input: local.jolokia_agent_misc not supported
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.kernel started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.kernel_vmstat started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.linux_sysctl_fs started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.mem started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.net started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.netstat started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.nfsclient started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.processes started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.self_metrics started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.sockstat started
2024/12/13 02:44:41 metrics_agent.go:318: I! input: local.system started
2024/12/13 02:44:41 agent.go:46: I! [*agent.MetricsAgent] started
2024/12/13 02:44:41 agent.go:46: I! [*agent.IbexAgent] started
2024/12/13 02:44:41 agent.go:49: I! agent started
2024/12/13 02:44:41 heartbeat.go:19: I! ibex agent start rolling request Server.Report.
2024/12/13 02:44:42 cli.go:84: I! choose server: 后端节点IP:20090, duration: 0ms
```