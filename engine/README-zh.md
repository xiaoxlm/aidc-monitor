# 说明
这是各类收集引擎模块。

这里面我们使用 prometheus 作为指标收集引擎；使用 loki 日志收集引擎。
>我们模拟是在同一个内网环境，即所有节点都是 ping 通。


# 修改目录权限
进入目录请先修改目录权限:
```shell
$ sudo chown -R nobody:nogroup data/ # 修改prometheus 数据目前的权限

$ sudo chown -R  10001:10001 loki/data/ # 修改 loki 数据目录的权限
```

# 修改 prometheus 的配置
在目录中修改 `prometheus.yml` 配置文件。

用户只需要在标有注释 `# 修改处` 的地方修改对应的值即可, 也就是 "opentelemetry-collector节点ip"！

# 启动
我们不需要修改docker-compose.yaml里面的内容，直接启动即可:
```shell
$ docker-compose -f docker-compose.yaml up -d

✔ Container prometheus  Started  0.3s
✔ Container engine_loki Started  0.4s
```

# 验证 prometheus 启动成功
我们在浏览器中输入 http://节点ip:9090，看到页面即启动成功。

# 验证 loki 启动成功
与 exporter 验证类似。能看到 loki 内部暴露的指标即启动成功。
```shell
curl http://节点:3100/metrics
```

## 补充
还有一种方式即通过 grafana 集成这两个引擎，然后通过 Explore 查询来验证。