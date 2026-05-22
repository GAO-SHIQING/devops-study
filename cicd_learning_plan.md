# CI/CD（GitHub Actions）零基础学习路线

---

## 一、前置条件

- 已掌握 Git 基本操作（clone / commit / push / branch / merge）
- 已掌握 Docker（镜像构建、Dockerfile、Compose）
- 有一个 GitHub 账号和基本的 GitHub 使用经验

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ 概念与基础    │    │ 构建与测试    │    │ Docker & 部署 │    │ 进阶实战      │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：概念与基础 — 跑通第一个 Pipeline

**目标：理解 CI/CD 理念，能写一个从 push 到 test 的 workflow**

#### Day 1：什么是 CI/CD

| 主题 | 内容 |
|------|------|
| 持续集成 (CI) | 频繁合并代码、自动构建、自动测试，尽早发现集成问题 |
| 持续交付 (CD) | CI 通过后自动将代码部署到 staging/production |
| 持续部署 | 更进一步：每次通过 CI 都自动上线（无需人工审批） |
| 传统发布 vs CI/CD | 手动构建 → 手动测试 → 手动部署 → 出问题回滚（对比自动化流水线） |

**练习：** 阅读一个真实开源项目的 GitHub Actions 配置（如 fastapi/fastapi 的 `.github/workflows/`）

#### Day 2：GitHub Actions 核心概念

| 概念 | 说明 |
|------|------|
| Workflow | 一个完整的自动化流程，定义在 `.github/workflows/*.yml` |
| Event / Trigger | 什么触发了 workflow：`push`、`pull_request`、`schedule`、`workflow_dispatch` |
| Job | 一组在同一个 Runner 上执行的 Step，默认并行运行 |
| Step | 最小的执行单元：可以是 `run`（shell 命令）或 `uses`（复用 action） |
| Runner | 执行 Job 的机器：GitHub 托管（ubuntu/macos/windows）或自建 |

**练习：** 在浏览器里打开你的仓库 → Actions 标签页，浏览 GitHub 推荐的工作流模板

#### Day 3：第一个 Workflow

YAML 语法速览：

| 要点 | 说明 |
|------|------|
| 缩进 | 2 空格，不能混用 Tab |
| 键值 | `key: value` |
| 列表 | `- item1` / `- item2` |
| 多行字符串 | `\|` (保留换行) 或 `>` (折叠成一行) |
| 表达式 | `${{ <expression> }}` |

**练习：** 创建一个仓库，在 `.github/workflows/ci.yml` 中写出一个最小 workflow：

```yaml
name: CI
on: push
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello CI/CD!"
```

push 后在 Actions 标签页观察运行结果。

#### Day 4：checkout 与多 Step 编排

| 主题 | 说明 |
|------|------|
| `actions/checkout` | 将仓库代码拉取到 Runner |
| Step 执行顺序 | 同 Job 的 Step 串行执行 |
| Step 条件 | `if: success()` / `if: failure()` / `if: always()` |
| `GITHUB_WORKSPACE` | Runner 上代码存放的路径 |

**练习：** 写一个 workflow：checkout 代码 → 列出文件 → 运行 `python --version` → 打印当前时间

#### Day 5：Runner 环境与 setup 类 Action

| 主题 | Action / 命令 |
|------|---------------|
| Python | `actions/setup-python@v5` |
| Node.js | `actions/setup-node@v4` |
| 系统信息 | `uname -a` / `cat /etc/os-release` |
| 预装软件 | Ubuntu Runner 已预装 Python、Node、Docker 等 |

**练习：** 写一个 workflow，用 `setup-python` 指定 Python 3.12，然后运行一段 Python 脚本

#### Day 6：多 Job 与依赖

| 主题 | 说明 |
|------|------|
| `jobs.<id>.needs` | 指定依赖，控制 Job 串行执行 |
| 默认并行 | 不设置 needs 的 Job 并行跑 |
| 跨 Job 传递数据 | 初步了解 artifacts（第2周深入） |

**练习：** 创建 3 个 Job：
```
lint（Python 代码格式检查）
  ├──▶ test（运行 pytest）
  └──▶ type-check（运行 mypy）
```
lint 先跑，成功后 test 和 type-check 并行跑。

#### Day 7：第一周综合练习

为目标仓库创建完整的 **push 触发 → 环境准备 → 代码检查 → 单元测试** 流程：

```
push 触发
  ├── Job: lint
  │     ├── checkout
  │     ├── setup-python
  │     └── ruff check .
  ├── Job: test (依赖 lint)
  │     ├── checkout
  │     ├── setup-python (Python 3.10 / 3.11 / 3.12 matrix)
  │     └── pytest
  └── Job: format (依赖 lint)
        ├── checkout
        └── ruff format --check .
```

---

### 第2周：构建与测试

**目标：掌握 Matrix Build、缓存、Artifacts、Secrets 管理**

#### Day 8：Matrix Strategy（多版本矩阵）

| 主题 | 说明 |
|------|------|
| 语法 | `strategy.matrix.<key>: [v1, v2, v3]` |
| 组合爆炸 | 多个维度的笛卡尔积（如 3 个 Python × 2 个 OS = 6 个 Job） |
| `fail-fast` | 默认 true，一个失败立即取消其余；设为 false 独立运行 |
| `include` / `exclude` | 添加或排除特定组合 |

**练习：** 写一个 matrix 测试：Python 3.10/3.11/3.12 × Ubuntu/macOS，在每个组合中运行 pytest

#### Day 9：依赖缓存

| 主题 | 说明 |
|------|------|
| `actions/cache@v4` | 缓存依赖，加速后续运行 |
| cache key | `key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}` |
| 命中 vs 未命中 | 命中时跳过安装，直接恢复缓存目录 |
| `restore-keys` | 部分匹配的降级缓存 |

**练习：** 给昨天的 matrix 测试加上 pip 依赖缓存，对比首次运行和缓存命中后的耗时差异

#### Day 10：Artifacts（构建产物）

| 主题 | 说明 |
|------|------|
| `actions/upload-artifact` | 上传构建产物（可跨 Job 传递） |
| `actions/download-artifact` | 下载之前 Job 上传的产物 |
| 保留天数 | 默认 90 天，可设置 `retention-days` |
| 典型场景 | 构建产物 → 部署时下载使用 |

**练习：** 在 Job A 中生成一个 Markdown 报告（如 `pytest --md report.md`），upload artifact；在 Job B 中 download 并展示内容

#### Day 11：Secrets 与 Variables 管理

| 主题 | 说明 |
|------|------|
| 仓库级 Secrets | Settings → Secrets and variables → Actions |
| 使用方式 | `${{ secrets.MY_SECRET }}`，日志中会自动屏蔽 |
| Variables | 非敏感配置，如 `DEPLOY_REGION` |
| 环境级 Secrets | 按 environment 隔离（如 dev / prod 各有自己的 `API_KEY`） |

**练习：** 在仓库中设置 `TEST_TOKEN` secret，写一个 workflow 打印它的长度（不要打印原始值）

#### Day 12：Pull Request 触发与 Status Check

| 主题 | 说明 |
|------|------|
| `on: pull_request` | PR 的 open、push、reopen 事件 |
| 分支过滤 | `branches: [main]` / `branches-ignore: [gh-pages]` |
| Status Check | PR 页面的绿色勾/红色叉，要求全部通过才能 merge（Branch Protection） |

**练习：** 写一个 PR 触发的 workflow（lint + test），去 Settings → Branches 设置 main 分支保护，要求 CI 全部通过才能 merge

#### Day 13：定时任务与手动触发

| 主题 | 说明 |
|------|------|
| `on: schedule` | `cron: "0 2 * * *"` (UTC 时间) |
| `on: workflow_dispatch` | 在 Actions 页面手动触发 |
| `inputs` | 手动触发时接受参数（环境、版本号等） |

**练习：** 写一个定时 workflow，每天凌晨 2 点运行一次完整测试；再加上手动触发选项，可指定运行哪些测试

#### Day 14：第二周综合练习

构建一个完整的 Python 包 CI 流水线：

```
push / PR 触发
  ├── Job: matrix-test
  │     ├── strategy: Python 3.10/3.11/3.12 × ubuntu/macos
  │     ├── pip 缓存
  │     ├── 运行 pytest
  │     └── upload coverage artifact
  ├── Job: lint
  │     └── ruff check + mypy
  ├── Job: build
  │     ├── 构建 Python wheel 包
  │     └── upload wheel artifact
  └── Job: coverage-report (依赖 test)
        ├── download coverage artifacts
        └── 生成 HTML coverage 报告 + upload
```

---

### 第3周：Docker & 部署

**目标：在 CI/CD 中构建镜像、推送到 Registry、部署到 Kubernetes**

#### Day 15：在 Actions 中构建 Docker 镜像

| 主题 | 说明 |
|------|------|
| `docker/build-push-action` | 官方 Action，构建并推送镜像 |
| 前置：`docker/login-action` | 登录到 Registry |
| 上下文路径 | `context: .` 指向 Dockerfile 所在目录 |
| build-args | 类似 `docker build --build-arg` |

**练习：** 写一个 workflow：checkout → build Docker 镜像（先只构建，不推送），确认构建成功

#### Day 16：推送到 Registry

| Registry | 说明 |
|----------|------|
| GitHub Container Registry (GHCR) | `ghcr.io/<user>/<image>` |
| Docker Hub | `docker.io/<user>/<image>` |
| 区别 | GHCR 与 GitHub 集成更好，无需额外注册 |

**练习：** 在 workflow 中构建镜像并推送到 GHCR：

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
```

#### Day 17：镜像 Tag 策略

| 策略 | 说明 |
|------|------|
| `latest` | 永远指向最新版本（dev 环境用） |
| commit SHA | `git rev-parse --short HEAD`，每个 commit 唯一 |
| 语义版本 | `v1.2.3`，tag push 触发 |
| `${{ github.ref_name }}` | 分支名或 tag 名 |

**练习：** 给 workflow 加上三类 tag：`latest`（main push）、`commit-<sha>`（每次 push）、`<tag>`（tag push 触发时）

#### Day 18：部署到 K8s（基础）

| 主题 | 说明 |
|------|------|
| 前置 | 已有 K8s 集群（可用 minikube 或 kind） |
| `actions-hub/kubectl` | 在 Runner 上配置 kubectl |
| kubeconfig | 通过 Secret 存储 `KUBE_CONFIG` |
| 滚动更新 | `kubectl set image deployment/app app=ghcr.io/x/y:latest` |

**练习：** 创建一个 K8s Deployment + Service YAML，在 CI 中把新镜像版本写入 manifest 并 `kubectl apply`

#### Day 19：多环境部署（dev / staging / prod）

| 主题 | 说明 |
|------|------|
| Environment | Settings → Environments，每个环境独立的 Secrets 和审批人 |
| `environment: <name>` | 在 Job 级别指定 |
| 保护规则 | Required reviewers、Wait timer、Branch restriction |
| 命名约定 | 一个 environment = 一个 K8s namespace |

**练习：** 创建 dev 和 staging 两个 Environment，配置各自的 KUBE_CONFIG secret，workflow 根据分支部署到不同环境：
- main → dev（自动）
- tag push (v*) → staging（需要审批）

#### Day 20：Docker Layer 缓存加速

| 主题 | 说明 |
|------|------|
| Registry cache | `cache-from: type=registry` / `cache-to: type=registry` |
| 原理 | 将已构建的 layer 推送到 Registry，下次 build 只构建变更层 |
| GHA cache backend | `cache-from: type=gha` / `cache-to: type=gha`（不需要 Registry） |

**练习：** 对比三种场景的构建时间：无缓存、GHA cache backend、Registry cache

#### Day 21：第三周综合练习

构建一个完整的"代码提交 → 镜像 → 部署"流水线：

```
push main / PR
  ├── Job: test (同第2周)
  ├── Job: lint
  └── Job: build-and-push (依赖 test + lint)
        ├── 构建 Docker 镜像（layer 缓存）
        ├── tag: latest / commit-sha / branch-name
        └── push to GHCR

tag push (v*)
  └── Job: deploy-staging (依赖 build-and-push, environment: staging)
        ├── 拉取 KUBE_CONFIG from secrets
        ├── kubectl set image → 更新 staging 集群
        └── kubectl rollout status → 等待部署完成
```

---

### 第4周：进阶与实战

**目标：Reusable Workflow、审批门、通知、完整项目实战**

#### Day 22：Reusable Workflows（可复用工作流）

| 主题 | 说明 |
|------|------|
| 定义 | `on: workflow_call`，接受 inputs / secrets |
| 调用 | `uses: <org>/<repo>/.github/workflows/<file>@<ref>` |
| 参数 | `inputs`（string/boolean/number）和 `secrets`（敏感值） |
| DRY 原则 | 多个仓库共享同一套 lint/test/build 逻辑 |

**练习：** 把之前的 test + lint workflow 抽成 reusable workflow，在另一个仓库中引用它

#### Day 23：Custom Actions

| 类型 | 说明 |
|------|------|
| Composite Action | 在 `action.yml` 中组合多个 Step（最轻量） |
| JavaScript Action | 用 Node.js 写 Action 逻辑 |
| Docker Action | 在 Docker 容器中运行任意代码 |

**练习：** 写一个 Composite Action：接受 Python 版本和包名作为 input，自动 setup → install → pytest → 输出结果

#### Day 24：通知集成

| 通道 | Action |
|------|--------|
| Slack | `slackapi/slack-github-action` |
| 企业微信 / 飞书 | webhook + `curl` |
| GitHub Issue | `actions/github-script` 创建 Issue |
| Email | 通过第三方邮件 API |

**练习：** 在 workflow 的末尾加上通知 Step：成功发 Slack 通知（或飞书 webhook），失败创建 GitHub Issue

#### Day 25：CI/CD 安全

| 主题 | 说明 |
|------|------|
| OIDC 认证 | 不用长期 Token，短时身份联邦（如 AWS / GCP 部署） |
| `permissions` | 在 workflow 或 job 级别最小化权限 |
| 第三方 Action 风险 | pin 到 commit SHA 而非 tag，审核代码 |
| Secret 排查 | 避免在日志中打印 secret（GitHub 自动屏蔽，但 echo 可能绕过） |

**练习：** 审查你之前所有 workflow 的 `permissions` 设置，把默认的 `write-all` 收紧为仅需要的权限

#### Day 26：成本优化

| 主题 | 说明 |
|------|------|
| Runner 选择 | ubuntu-latest 免费额度最多 |
| 跳过不必要的运行 | `paths-ignore` 在 on.push 中排除文档变更 |
| Cache 清理 | 过期的 cache 自动清理，但显式删除可节省空间 |
| 并发控制 | `concurrency` 取消同分支的旧运行 |

**练习：** 给你最常用的仓库设置 `concurrency` 规则，并配置 `paths-ignore` 跳过 README 等非代码变更

#### Day 27-28：第四周综合练习

为一个真实项目（可以是你的 Python 学习项目）搭建完整的 CI/CD 流水线：

```
GitHub 仓库 CI/CD 全景：

1. PR 阶段
   ├── lint (ruff + mypy)
   ├── test (matrix: 3 Python × 2 OS, 带缓存)
   └── security scan (pip-audit / safety)

2. Merge 到 main 后
   ├── build Docker image (layer 缓存)
   ├── push to GHCR (latest + commit-sha)
   ├── deploy to dev K8s (自动)
   └── 通知 Slack "dev 已更新"

3. 发版 Tag (v*)
   ├── build Docker image
   ├── push (latest + tag + commit-sha)
   ├── deploy to staging K8s (需要审批)
   └── 跑一遍 E2E 测试

4. 每日定时
   ├── 运行全部测试 (schedule cron)
   └── 风险：失败创建 Issue
```

---

## 四、里程碑检查点

```
Week 1 结束：✓ 能写基本的 push/PR 触发 workflow，理解 Job/Step/Runner 关系
Week 2 结束：✓ 能配置 matrix build、缓存、artifacts、secrets，设置 Branch Protection
Week 3 结束：✓ 能在 CI 中构建 Docker 镜像并部署到 K8s，管理多环境
Week 4 结束：✓ 能编写 reusable workflow、custom action，搭建生产级 CI/CD 流水线
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 官方文档 | [docs.github.com/en/actions](https://docs.github.com/en/actions) —— 必读，示例丰富 |
| 市场 | [github.com/marketplace?type=actions](https://github.com/marketplace?type=actions) —— 寻找现成 Action |
| 学习仓库 | [github.com/sdras/awesome-actions](https://github.com/sdras/awesome-actions) —— Action 精选列表 |
| 在线练习 | [github.com/skills](https://github.com/skills/) —— GitHub 官方互动教程 |
| 参考项目 | 大项目的 `.github/workflows/` 目录是最好的学习材料 |
| CI/CD 理念 | 《持续交付》(Jez Humble) —— 经典著作，理解"为什么" |
