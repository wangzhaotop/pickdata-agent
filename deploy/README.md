# 中间件部署

该目录通过 Docker Compose 启动 PickData Copilot 所需的 MySQL、Redis、Milvus 和 RocketMQ。中间件均加入 `pickdata-middleware` 网络。

## 启动

```powershell
cd deploy
Copy-Item .env.example .env
```

修改 `.env` 中的密码后，按需选择启动范围。

```powershell
# MySQL + Redis
docker compose up -d

# MySQL + Redis + Milvus 依赖
docker compose --profile vector up -d

# MySQL + Redis + RocketMQ
docker compose --profile mq up -d

# 全部中间件
docker compose --profile vector --profile mq up -d
```

Milvus 单机模式会同时启动 Milvus、etcd 和 MinIO。RocketMQ 会同时启动 NameServer 和 Broker。

## 端口

| 服务 | 地址 | 用途 |
|---|---|---|
| MySQL | `127.0.0.1:3306` | 业务库与样例数仓 |
| Redis | `127.0.0.1:6379` | 锁、幂等与缓存 |
| Milvus | `127.0.0.1:19530` | 指标目录向量检索 |
| Milvus Health | `http://127.0.0.1:9091/healthz` | 健康检查 |
| MinIO API | `http://127.0.0.1:9000` | Milvus 对象存储 |
| MinIO Console | `http://127.0.0.1:9001` | MinIO 管理页面 |
| RocketMQ NameServer | `127.0.0.1:9876` | 客户端路由 |
| RocketMQ Broker | `127.0.0.1:10911` | Broker 通信 |

账号、密码与端口统一在 `.env` 中维护。`.env` 已被 Git 忽略，不要提交真实密钥。

## 检查与排障

```powershell
docker compose ps
docker compose logs -f mysql redis
docker compose --profile vector logs -f etcd minio milvus
docker compose --profile mq logs -f rocketmq-namesrv rocketmq-broker
```

```powershell
docker compose exec mysql mysqladmin ping -uroot -p
docker compose exec redis redis-cli ping
docker compose --profile vector exec milvus curl -f http://localhost:9091/healthz
```

## 停止

```powershell
# 停止容器，保留数据卷
docker compose --profile vector --profile mq down

# 连同 MySQL、Redis、Milvus 和 RocketMQ 数据一起删除
docker compose --profile vector --profile mq down -v
```

`broker.conf` 中的 `brokerIP1` 默认使用 `127.0.0.1`，适用于应用运行在 Docker 宿主机上的开发模式。若后续把应用容器化并加入 `pickdata-middleware` 网络，应改为 `rocketmq-broker`，并同步调整客户端配置。
