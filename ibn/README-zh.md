# 说明
安装 xtrace 监控系统所依赖的 ibn 系统

# 先启动 arangodb
```shell
docker run -d -p 8529:8529 -v ./backup_xtrace_dev:/etc/backup_xtrace_dev -e ARANGO_ROOT_PASSWORD='123456' --name arangodb-instance arangodb:3.9.12
```

然后登录 arangodb web-ui 创建 `xtrace` 用户和 `xtrace-dev` database

# 导入备份数据
arangodb 配置外用户和数据库后，执行下面操作导入备份数据:
```shell
1. 进入 arangodb 容器: docker exec -it  arangodb-instance sh
2. 创建用户 xtrace 和 数据库 xtrace-dev
3. 执行命令: arangorestore --server.endpoint tcp://localhost:8529 --server.username xtrace --server.password 123456 --server.database xtrace-dev  --input-directory /etc/backup_xtrace_dev 
```

# 启动 ibn 进程:
```shell
docker-compose -f ./docker-compose.yml up -d
```