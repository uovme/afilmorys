# 1Panel 部署指南

本指南介绍如何通过 **1Panel** 在 VPS 上部署 Afilmory，使用 1Panel 已有的 PostgreSQL 和 Redis 服务。

## 前提条件

- VPS 已安装 [1Panel](https://1panel.cn/)
- 1Panel 中已安装 **PostgreSQL** 和 **Redis**
- 已安装 **Docker** 和 **Docker Compose**（1Panel 会自动安装）

## 部署步骤

### 1. 获取 1Panel 服务信息

SSH 登录到 VPS，运行以下命令获取关键信息：

```bash
# 查看 PostgreSQL 和 Redis 容器名
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}" | grep -E "postgres|redis"

# 查看 Docker 网络名
docker network ls
```

记录以下信息（后续配置需要用到）：

| 信息 | 示例值 | 你的值 |
|------|--------|--------|
| PostgreSQL 容器名 | `1panel-postgresql` | |
| Redis 容器名 | `1panel-redis` | |
| Docker 网络名 | `1panel-network` | |

> **提示**：如果不确定网络名，可以运行：
> ```bash
> docker inspect 你的PostgreSQL容器名 | grep -A5 Networks
> ```

### 2. 创建数据库

在 1Panel 管理后台中：

1. 进入 **数据库** → **PostgreSQL**
2. 点击 **创建数据库**
   - 数据库名：`afilmory`
   - 用户名：`afilmory`
   - 密码：设置一个强密码（记住它！）

### 3. 拉取项目代码

```bash
git clone https://github.com/uovme/afilmorys.git
cd afilmorys
```

### 4. 配置环境变量

```bash
# 复制模板文件
cp .env.docker.example .env.docker

# 编辑环境变量
nano .env.docker
```

**必须修改的配置项**：

```env
# 数据库连接 — 替换为你的实际值
DATABASE_URL=postgresql://afilmory:你的数据库密码@你的PostgreSQL容器名:5432/afilmory

# Redis 连接 — 替换为你的实际值
# 无密码: redis://容器名:6379/0
# 有密码: redis://:密码@容器名:6379/0
REDIS_URL=redis://你的Redis容器名:6379/0

# 加密密钥 — 必须替换！
# 运行 openssl rand -hex 32 生成
CONFIG_ENCRYPTION_KEY=这里填生成的密钥
```

**示例**（假设容器名为 `1panel-postgresql` 和 `1panel-redis`）：

```env
DATABASE_URL=postgresql://afilmory:MyStr0ngP@ss@1panel-postgresql:5432/afilmory
REDIS_URL=redis://1panel-redis:6379/0
CONFIG_ENCRYPTION_KEY=a1b2c3d4e5f6...（64 位十六进制字符串）
```

### 5. 配置 Docker 网络

编辑 `docker-compose.yml`，确认网络名与 1Panel 一致：

```bash
nano docker-compose.yml
```

修改底部的网络配置（如果你的 1Panel 网络名不是 `1panel-network`）：

```yaml
networks:
  1panel-network:       # ← 改成你实际的网络名
    external: true
```

同时确认 `core` 服务的 `networks` 也匹配：

```yaml
    networks:
      - 1panel-network  # ← 同上
```

### 6. 构建并启动

```bash
docker compose up -d --build
```

首次构建可能需要 5-10 分钟，取决于 VPS 性能。

### 7. 验证部署

```bash
# 查看容器状态
docker ps | grep afilmory

# 查看启动日志
docker logs -f afilmory_core
```

**正常日志输出**：

```
[entrypoint] Running database migrations...
[CLI:DB_MIGRATE] Applying database migrations...
[CLI:DB_MIGRATE] Database migrations applied successfully
[entrypoint] Starting application...
[DB] Database connection established successfully
[Redis] Redis connecting...
[Redis] Redis connection established successfully
Hono HTTP application started on http://0.0.0.0:1841. +XXXms
```

按 `Ctrl+C` 退出日志查看。

### 8. 配置反向代理（推荐）

在 1Panel 中为 Afilmory 站点设置反向代理：

1. 进入 **网站** → **创建站点** 或编辑已有站点
2. 设置 **反向代理**：
   - 代理地址：`http://127.0.0.1:1841`
3. （可选）配置 SSL 证书以启用 HTTPS

配置完成后即可通过域名访问 Afilmory。

## 常用管理命令

```bash
# 查看日志
docker logs -f afilmory_core

# 重启服务
docker compose restart core

# 停止服务
docker compose down

# 更新部署（拉取最新代码后）
git pull
docker compose up -d --build

# 重置超级管理员密码
docker exec afilmory_core node ./dist/main.js reset-superadmin-password
```

## 常见问题

### Core 容器不断重启

```bash
docker logs afilmory_core
```

根据日志判断原因：

| 日志关键词 | 原因 | 解决方案 |
|-----------|------|---------|
| `exec format error` 或 `not found` | 脚本换行符问题 | 重新 `git pull` 后 `docker compose up --build` |
| `DATABASE_URL is required` | 环境变量缺失 | 检查 `.env.docker` 文件是否存在且配置正确 |
| `connection refused` | 网络不通 | 确认 `docker-compose.yml` 中的网络名与 1Panel 一致 |
| `password authentication failed` | 数据库密码错误 | 在 1Panel 数据库管理中确认用户名和密码 |
| `ECONNREFUSED` (Redis) | Redis 不可达 | 确认 Redis 容器名和网络配置 |

### 无法访问网页

1. 检查容器是否正常运行：`docker ps | grep afilmory`
2. 测试端口是否监听：`curl http://127.0.0.1:1841/`
3. 检查防火墙是否放行端口 1841（如直接用 IP 访问）
4. 如使用反向代理，检查反向代理配置是否正确

### 修改端口

编辑 `.env.docker`，修改 `APP_PORT`：

```env
APP_PORT=8080  # 宿主机映射端口
PORT=1841      # 容器内端口（通常不需要改）
```

然后重启：`docker compose up -d`
