# 说明
这是监控平台模块。整个监控平台的展示与操作都在这个模块中完成。

由于所有账号密码都已经设置好，用户可以无需修改直接启动。
>如果要修改账号密码，请联系技术人员修改对应处即可



# 先中间件启动
进入目录后，不需要修改docker-compose.yaml里面的内容，直接启动即可:
```shell
$ docker-compose -f docker-compose.yaml up -d mysql redis

✔ Container n9e_redis       Started   0.5s
✔ Container n9e_mysql       Started   0.5s
```

## 导入sql
mysql 启动成功后，我们需要导入一些准备的数据。

我们可以在本地使用 mysql 客户端工具如 navicat 连上数据库后，将 `xtrace.sql` 导入即可。

然后我们就可以在页面看到已经配置好的数据。

# 再启动系统
# 修改配置
我们需要修改后端系统的配置文件 `n9e-config/config.toml` 当中的两处信息，即mysql节点ip, redis节点ip:
```toml
DSN = "root:uWXf87plmQGz8zMM@tcp(mysql节点ip:3306)/n9e_v6?charset=utf8mb4&parseTime=True&loc=Local&allowNativePasswords=true"

[Redis]
Address = "redis节点ip:6379"
```

然后启动
```shell
$ docker-compose -f docker-compose.yaml up -d n9e tracex-ui

✔ Container n9e             Started   0.8s
✔ Container tracex-ui       Started   0.8s
```

## 验证启动成功
监控系统启动后， 我们在浏览器中输入 "http://节点ip:8081"，看到页面即我们部署成功。
初始账号密码是
```shell
root # 账号
root.2020 # 密码
```