# FastAPI 后端部署文档

---

## 技术栈

| 层 | 技术 |
|---|------|
| 框架 | FastAPI (Python 3.10) |
| 包管理 | uv (astral-sh) |
| 进程管理 | supervisord |
| 数据库 | PostgreSQL 16 |
| 缓存 | Redis |
| 镜像仓库 | GitHub Container Registry (ghcr.io) |
| CI/CD | GitHub Actions |

---

## 镜像构建

### Dockerfile 多目标构建

```dockerfile
ARG SERVER_TYPE=fba_server

# 阶段 1: builder — uv 安装 Python 依赖
FROM ghcr.io/astral-sh/uv:python3.10-bookworm-slim AS builder
COPY . /fba
WORKDIR /fba
RUN uv sync --locked --no-default-groups --group server --no-install-project

# 阶段 2: base_server — 运行时基础层
FROM ghcr.io/astral-sh/uv:python3.10-bookworm-slim AS base_server
RUN apt-get install -y curl ca-certificates supervisor
COPY --from=builder /fba /fba
COPY --from=builder /usr/local /usr/local

# 阶段 3: 四个运行时 target
FROM base_server AS fba_server          # FastAPI 主服务 :8001
FROM base_server AS fba_celery_worker   # Celery Worker
FROM base_server AS fba_celery_beat     # Celery Beat
FROM base_server AS fba_celery_flower   # Celery Flower :8555

FROM ${SERVER_TYPE}
```

**四个镜像从同一个 base 层构建**，共用 Python 依赖层，总大小约 105MB/target。

### 手动构建（可选）

```bash
cd fastapi-best-architecture
docker build --build-arg SERVER_TYPE=fba_server -t fba_server .
```

---

## 部署架构

```
GitHub push main
  → Actions: matrix 构建 4 个镜像
  → push → ghcr.io/moi-igit/fastapi-demo:{fba_server|fba_celery_worker|fba_celery_beat|fba_celery_flower}
  → SSH 到 ECS
  → docker compose pull backend
  → docker compose up -d --force-recreate backend
```

> 目前 docker-compose 只关联 `fba_server`，celery 系列镜像已构建但暂未编排。

---

## 服务器配置

### 目录结构

```
/opt/apps/project-a/
├── docker-compose.yml       # 统一编排
├── backend-env.server       # .env 环境变量（绑定到 /fba/backend/.env）
├── backend-src/             # 源码（仅首次初始化用）
├── logs/fba/                # 应用日志
├── fba-data/
│   ├── postgres/            # PG 数据持久化
│   └── redis/               # Redis 数据持久化
└── ...
```

### docker-compose.yml

```yaml
services:
  backend:
    image: ghcr.io/moi-igit/fastapi-demo:fba_server
    container_name: project-a-backend
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./backend-env.server:/fba/backend/.env:ro
      - ./logs/fba:/var/log/fba
      - project-a-static:/fba/backend/app/static
      - project-a-upload:/fba/backend/static/upload
    restart: unless-stopped
    depends_on:
      - db
      - redis
    command:
      - bash
      - -c
      - |
        wait-for-it -s db:5432 -s redis:6379 -t 300
        supervisord -c /etc/supervisor/supervisord.conf
        supervisorctl restart
    networks:
      - project-a-internal

  db:
    image: postgres:16
    container_name: project-a-db
    environment:
      POSTGRES_DB: fba
      POSTGRES_PASSWORD: 123456
      TZ: Asia/Shanghai
    volumes:
      - ./fba-data/postgres:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - project-a-internal

  redis:
    image: redis:latest
    container_name: project-a-redis
    command: redis-server --appendonly yes
    volumes:
      - ./fba-data/redis:/data
    restart: unless-stopped
    networks:
      - project-a-internal

networks:
  project-a-internal:
    name: project-a-internal

volumes:
  project-a-static:
  project-a-upload:
```

> `project-a-internal` 是内部网络，前端通过同一网络访问 `backend:8001`。

### 环境变量 (`backend-env.server`)

```bash
ENVIRONMENT=prod
DATABASE_TYPE=postgresql
DATABASE_HOST=db               # docker-compose 服务名
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=123456
REDIS_HOST=redis
REDIS_PORT=6379
TOKEN_SECRET_KEY=your-secret-key-here
CORS_ALLOWED_ORIGINS=["http://47.109.155.75","*"]
```

---

## GitHub Actions (`deploy.yml`)

```yaml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    strategy:
      matrix:
        target:
          - fba_server
          - fba_celery_worker
          - fba_celery_beat
          - fba_celery_flower

    steps:
      - uses: actions/checkout@v4
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push ${{ matrix.target }}
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          build-args: SERVER_TYPE=${{ matrix.target }}
          tags: ghcr.io/${{ github.repository }}:${{ matrix.target }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.2.1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          password: ${{ secrets.SSH_PASSWORD }}
          command_timeout: 5m
          script: |
            cd /opt/apps/project-a
            timeout 180 docker compose pull backend 2>&1 || echo "timeout"
            docker compose up -d --force-recreate backend
```

---

## GitHub Secrets（需配置）

| Secret | 值 |
|--------|---|
| `SSH_HOST` | `47.109.155.75` |
| `SSH_USER` | `root` |
| `SSH_PASSWORD` | 服务器 root 密码 |

---

## 手动部署

```bash
# 服务器上执行
cd /opt/apps/project-a
docker compose pull backend
docker compose up -d --force-recreate backend
```

---

## 验证

```bash
# 容器状态
docker ps | grep project-a

# 健康检查
curl http://47.109.155.75/project-a/api/v1/auth/login

# 查看日志
docker logs --tail 50 project-a-backend
docker logs --tail 50 project-a-db
```

---

## 数据库连接信息

| 项目 | 值 |
|------|-----|
| 容器名 | `project-a-db` |
| 镜像 | `postgres:16` |
| 数据库 | `fba` |
| 用户 | `postgres` |
| 密码 | `123456` |
| 端口 | `5432`（容器内） |

> ⚠️ 密码 `123456` 仅测试用，生产环境必须修改。

### 连接方式

```bash
# 从服务器直接进入
docker exec -it project-a-db psql -U postgres -d fba

# 应用内连接（容器间）
# Host: db  (docker-compose 服务名)
# Port: 5432
# User: postgres
# Password: 123456
# Database: fba
```

---

## 数据库操作

```bash
# 进入 PostgreSQL
docker exec -it project-a-db psql -U postgres -d fba

# 查看表
\dt

# 查询用户
SELECT * FROM "user";
```

---

## 环境变量说明

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ENVIRONMENT` | 运行环境 | `prod` |
| `DATABASE_TYPE` | 数据库类型 | `postgresql` |
| `DATABASE_HOST` | 数据库地址 | `db`（容器名） |
| `DATABASE_PORT` | 数据库端口 | `5432` |
| `DATABASE_USER` | 数据库用户 | `postgres` |
| `DATABASE_PASSWORD` | 数据库密码 | `123456` |
| `REDIS_HOST` | Redis 地址 | `redis` |
| `REDIS_PORT` | Redis 端口 | `6379` |
| `TOKEN_SECRET_KEY` | JWT 签名密钥 | 生产环境必须修改 |
| `CELERY_RABBITMQ_HOST` | RabbitMQ 地址 | （可选） |

---

## 重启与故障排查

```bash
# 重启后端
docker compose up -d --force-recreate backend

# 重启所有服务
docker compose down && docker compose up -d

# 清空数据库（危险！）
docker compose down -v
rm -rf /opt/apps/project-a/fba-data/postgres/*
docker compose up -d
```
