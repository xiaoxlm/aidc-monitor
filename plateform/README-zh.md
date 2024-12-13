# 说明
这是监控平台模块。整个监控平台的展示与操作都在这个模块中完成。

由于所有账号密码都已经设置好，用户可以无需修改直接启动。
>如果要修改账号密码，请联系技术人员修改对应处即可

# 启动
进入目录后，不需要修改docker-compose.yaml里面的内容，直接启动即可:
```shell
docker-compose -f docker-compose.yaml up -d

✔ Container n9e_redis       Started   0.5s
✔ Container n9e_mysql       Started   0.5s
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


# 导入 sql
启动成功后，我们需要导入一些准备的数据。 

我们可以在本地使用 mysql 客户端工具如 navicat 连上数据库后，将 `xtrace.sql` 导入即可。

然后我们就可以在页面看到已经配置好的数据。