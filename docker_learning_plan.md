# Docker 零基础学习路线

---

## 一、前置条件

- 已掌握基本的命令行操作（cd / ls / mkdir / rm 等）
- 有一个可用的 Linux 环境（或 macOS / Windows WSL2）

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ 容器思维      │    │ Dockerfile    │    │ Compose       │    │ 实战综合      │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：容器思维 & 基本操作

**目标：理解"为什么需要 Docker"，能跑起容器**

#### Day 1：问题与概念

| 主题 | 内容 |
|------|------|
| 没有容器时的问题 | "在我机器上能跑"、环境不一致、依赖地狱 |
| 虚拟机 vs 容器 | 资源开销对比、启动速度、隔离级别 |
| Docker 架构 | Client-Server 模型、docker daemon、registry |

**练习：** 安装 Docker，跑通 `docker run hello-world`

#### Day 2：镜像与容器

| 主题 | 说明 |
|------|------|
| 镜像（Image） | 只读模板，分层存储，写时复制 |
| 容器（Container） | 镜像的运行实例，有独立的文件系统、网络、进程 |
| Docker Hub | 官方镜像仓库，搜索和拉取镜像 |

**练习：**
```bash
docker pull nginx
docker pull python:3.12
docker images
docker run -d --name my-nginx -p 8080:80 nginx
curl localhost:8080
docker ps
docker stop my-nginx
docker rm my-nginx
```

#### Day 3：容器的生命周期

| 主题 | 命令 |
|------|------|
| 创建与运行 | `docker run` / `docker create` |
| 启停 | `docker start` / `docker stop` / `docker restart` |
| 查看 | `docker ps` / `docker ps -a` / `docker logs` |
| 删除 | `docker rm` / `docker rmi` |
| 进入容器 | `docker exec -it <container> bash` |

**练习：** 完整操作一遍容器的创建→运行→进入→停止→删除流程

#### Day 4：端口映射、卷挂载、环境变量

| 主题 | 命令/参数 |
|------|-----------|
| 端口映射 | `-p 主机端口:容器端口` |
| 卷挂载 | `-v 主机路径:容器路径`（bind mount） |
| 环境变量 | `-e KEY=VALUE`，`--env-file` |

**练习：**
```bash
# 端口映射
docker run -d -p 3000:80 nginx

# 卷挂载：修改本地文件 → 容器内立即生效
echo "<h1>Hello Docker</h1>" > /tmp/index.html
docker run -d -p 8081:80 -v /tmp/index.html:/usr/share/nginx/html/index.html nginx

# 环境变量
docker run -e MYSQL_ROOT_PASSWORD=secret -d mysql:8
```

#### Day 5：网络与调试

| 主题 | 说明 |
|------|------|
| 网络模式 | bridge（默认）/ host / none |
| 自定义网络 | `docker network create` |
| 容器互访 | 同网络下用容器名互通 |
| 查看信息 | `docker inspect` / `docker stats` / `docker top` |

**练习：** 创建自定义网络，让两个容器（Python 脚本 + Redis）通过容器名互相访问

#### Day 6：镜像管理

| 主题 | 说明 |
|------|------|
| 镜像分层 | 理解 layer 概念、缓存复用 |
| tag 管理 | `docker tag`、版本命名规范 |
| 导入导出 | `docker save` / `docker load` / `docker export` |
| 清理空间 | `docker system prune`、`docker image prune` |

**练习：** 给镜像打多个 tag，理解 tag 只是指针

#### Day 7：第一周综合练习

启动以下环境，全部用 docker 命令完成：

```
Python 3.12 容器（挂载本地代码目录）
    ↕ 通信
Redis 容器（做缓存）
    ↕ 通信
MySQL 容器（持久化数据到本地卷）
```

验证：在 Python 容器里能 `ping redis`、`ping mysql`

---

### 第2周：Dockerfile — 构建自己的镜像

**目标：能将任意 Python 应用打包成镜像**

#### Day 8：第一个 Dockerfile

| 指令 | 说明 |
|------|------|
| `FROM` | 基础镜像 |
| `RUN` | 构建时执行命令 |
| `CMD` | 容器启动时的默认命令 |
| `COPY` | 从宿主机复制文件到镜像 |
| `WORKDIR` | 设置工作目录 |

**练习：** 写一个 Dockerfile，将一个 `print("hello")` 脚本打包运行

#### Day 9：更多指令

| 指令 | 说明 |
|------|------|
| `ADD` | 比 COPY 多了解压和 URL 支持（不推荐常用） |
| `EXPOSE` | 声明端口（文档作用，不实际映射） |
| `ENV` | 设置环境变量 |
| `ARG` | 构建参数 |
| `ENTRYPOINT` | 入口点，与 CMD 配合 |
| `VOLUME` | 声明匿名卷 |

**练习：** 对比 CMD 和 ENTRYPOINT 的不同行为

#### Day 10：构建实战 — Python 应用

**练习：** 将一个 Python 命令行 Todo 应用打包成镜像

Dockerfile 要点：
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "todo.py"]
```

构建并测试：
```bash
docker build -t todo-app:1.0 .
docker run --rm -v $(pwd)/data:/app/data todo-app:1.0
```

#### Day 11：构建优化

| 主题 | 说明 |
|------|------|
| layer 缓存 | 不常变的放前面（先 COPY requirements.txt） |
| `.dockerignore` | 排除不需要的文件 |
| 多阶段构建 | 编译和运行分离，减体积 |
| 非 root 运行 | `USER` 指令，安全实践 |
| 镜像瘦身 | slim/alpine 基础镜像、清理缓存 |

**练习：** 用多阶段构建优化一个 Python 镜像，对比前后体积

#### Day 12：最佳实践

| 实践 | 说明 |
|------|------|
| 一个容器一个进程 | 不要用 supervisord 管多个进程 |
| 日志输出到 stdout/stderr | 不要写日志文件 |
| 配置走环境变量 | 不要写死在代码里 |
| 数据走 volume | 不要存容器内 |
| 健康检查 | `HEALTHCHECK` 指令 |

**练习：** 对照清单，检查之前写的 Dockerfile 是否符合最佳实践

#### Day 13：镜像仓库

| 主题 | 说明 |
|------|------|
| Docker Hub | `docker login` / `docker push` / `docker pull` |
| 私有仓库 | `docker run -d -p 5000:5000 registry:2` |
| 自动化构建 | Docker Hub 关联 GitHub |

**练习：** 将自己构建的镜像推送到 Docker Hub

#### Day 14：第二周综合练习

将一个 Flask Web 应用完整容器化：

```
需求：
1. 多阶段构建，最终镜像 < 200MB
2. 非 root 运行
3. 带健康检查
4. 推送至 Docker Hub
5. 写 README 说明如何拉取和运行
```

---

### 第3周：Docker Compose — 多容器编排

**目标：能编排多服务应用**

#### Day 15：为什么需要 Compose

- 手动 `docker run` 启动多容器，命令太长、容易出错
- Compose 用 YAML 声明所有服务，一条命令全启动
- compose.yml 可以作为代码提交到 Git

**练习：** 对比手动启动 3 个容器 vs 用 compose 启动

#### Day 16：compose.yml 语法

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - ./app:/app
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

**练习：** 写一个 Python Web + Redis 的 compose 文件并跑起来

#### Day 17：Compose 命令

| 命令 | 说明 |
|------|------|
| `docker compose up` | 启动所有服务（`-d` 后台） |
| `docker compose down` | 停止并删除资源 |
| `docker compose ps` | 查看服务状态 |
| `docker compose logs` | 查看日志（`-f` 跟踪） |
| `docker compose exec` | 进入服务容器 |
| `docker compose build` | 重新构建镜像 |

#### Day 18：网络与依赖

| 主题 | 说明 |
|------|------|
| 默认网络 | Compose 自动创建网络，服务名即 DNS |
| `depends_on` | 启动顺序控制（不等健康检查） |
| 健康检查依赖 | `condition: service_healthy`（v3.9+） |
| 自定义网络 | 多网络拓扑 |

#### Day 19：卷与配置

| 主题 | 说明 |
|------|------|
| 命名卷 | compose 管理的卷，跨容器共享 |
| bind mount | 开发时直接挂载本地目录 |
| `env_file` | 从文件加载环境变量 |
| `secrets` | Swarm 模式下的密钥管理 |

#### Day 20：环境区分

| 主题 | 说明 |
|------|------|
| 多 compose 文件 | `docker compose -f base.yml -f dev.yml up` |
| `extends` | 继承服务定义 |
| profile | 按场景启用服务 |
| override | `docker-compose.override.yml` 自动合并 |

**练习：** 为同一项目配置 dev / prod 两种 compose 环境

#### Day 21：第三周综合练习

用 Compose 编排一个完整的 Python Web 应用：

```
服务清单：
├── web      Flask API（自己写的，Dockerfile 构建）
├── worker   Celery 异步任务（同一份代码，不同 CMD）
├── redis    缓存 + 消息队列
├── db       PostgreSQL（挂载卷持久化）
└── nginx    反向代理（挂载自定义配置）
```

---

### 第4周：进阶与实战

#### Day 22：Docker 与 CI/CD

| 主题 | 说明 |
|------|------|
| 在 GitHub Actions 中构建镜像 | CI 自动 build + push |
| 版本策略 | `latest` / `commit-sha` / `语义版本` |
| 缓存优化 | 利用 GitHub Actions cache 加速 |

#### Day 23：监控与日志

| 主题 | 说明 |
|------|------|
| `docker stats` | 资源使用监控 |
| 日志驱动 | json-file / syslog / fluentd |
| `docker events` | 实时事件流 |
| cAdvisor | 容器监控面板 |

#### Day 24：安全基础

| 主题 | 说明 |
|------|------|
| 镜像扫描 | `docker scout` 或 `trivy` |
| 非 root | 永远不要用 root 运行 |
| 只读文件系统 | `--read-only` |
| 能力限制 | `--cap-drop=ALL` |
| 密钥管理 | 不要把密码写进镜像/环境变量 |

#### Day 25：容器排障

| 场景 | 方法 |
|------|------|
| 容器起不来 | `docker logs`、`docker inspect` |
| 网络不通 | `docker network inspect`、进入容器 curl |
| 磁盘满了 | `docker system df`、`docker system prune` |
| 性能瓶颈 | `docker stats`、`docker top` |

#### Day 26-28：第四周综合练习

将之前学过的 Python 项目全部容器化：

1. **命令行 Todo** → 打包镜像 + 数据卷挂载
2. **Flask/FastAPI 应用** → Compose 编排 + 多环境配置
3. **爬虫项目** → 定时运行容器 + 结果输出到卷
4. 写一份 `DOCKER.md` 文档总结核心命令和踩坑记录

---

## 四、里程碑检查点

```
Week 1 结束：✓ 能熟练使用 docker run / ps / stop / exec 操作容器
Week 2 结束：✓ 能为任意 Python 项目编写优化的 Dockerfile
Week 3 结束：✓ 能用 Compose 编排多服务应用
Week 4 结束：✓ 能处理容器化过程中的安全、体积优化、排障问题
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 官方文档 | docs.docker.com（写得好，必看） |
| 在线练习 | Play with Docker（浏览器里玩 Docker） |
| 书籍 | 《Docker 技术入门与实战》（中文） |
| 参考 | Dockerfile 最佳实践（官方文档内） |
| 工具 | Dive（分析镜像分层，可视化体积） |
