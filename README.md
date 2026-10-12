# 共享 Redis

使用官方 `redis:8-alpine` 镜像，所有项目通过同一个密码连接。

## 目录

```text
redis/
├── docker-compose.yaml
├── .env
├── conf/
│   └── redis.conf
└── data/
```

Compose 将 `conf/redis.conf` 只读挂载到容器的配置文件路径，将 `data/` 挂载到容器的 `/data`。配置文件和数据都位于本服务目录。

## 启动

在本目录复制密码模板，编辑 `.env`，将示例密码替换为真实密码。可用 `openssl rand -hex 32` 生成密码，避免特殊字符转义。

```sh
cp .env.example .env
chmod 600 .env
docker compose up -d
docker compose ps
docker compose logs --tail=50 redis
```

`REDIS_PASSWORD` 是 Compose 读取的变量，随后通过 Redis 官方 `--requirepass` 参数设置密码。无需启动脚本，无需手动创建账号。修改密码后执行 `docker compose up -d` 重建容器即可生效。不要把真实 `.env` 发给别人或提交到 Git。

## 连接

连接地址：`<服务器 IP 或域名>:16379`。填写 `.env` 中的密码。需要用户名的客户端填写 `default`，其他客户端仅填写密码。连接使用普通 TCP，不启用 TLS。

云端安全组及服务器防火墙需允许 TCP 16379。Compose 的端口映射不负责修改云安全组。

同机 Docker 项目可以加入已有的 `shared-redis` 网络，通过 `redis:6379` 连接。其他项目的 Compose 声明该网络为 `external: true`，并将应用服务加入该网络即可。

## 数据和内存

AOF 每秒同步，RDB 每 15 分钟有写入时生成快照。文件保存在 `data/` 中，容器停止、重启、重建不会删除这些文件。AOF 每秒同步不能保证突然断电时最后一秒的数据不丢失。

Redis 内存上限为 128 MiB，容器总内存上限为 384 MiB。达到 Redis 内存上限后，新增写入可能报错，已有键不会被自动淘汰。

## 备份

先停止服务再备份整个数据目录，保证 AOF、RDB 一致：

```sh
mkdir -p backups
docker compose stop redis
tar -czf "backups/redis-$(date +%Y%m%d-%H%M%S).tar.gz" -C data .
docker compose start redis
```

恢复时先停止 Redis，保留现有 `data/` 的副本，再用备份替换完整数据目录，最后启动服务。不要直接混合新旧 AOF 文件。
