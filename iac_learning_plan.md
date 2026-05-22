# 基础设施即代码（IaC）学习路线

---

## 一、前置条件

- 已掌握 Docker 和 Kubernetes 基础
- 理解 CI/CD 基本概念（GitHub Actions / Jenkins）
- 有 Linux 命令行基础
- 有 SSH 基本概念（密钥对、远程登录）

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ Terraform     │    │ Terraform     │    │ Ansible       │    │ 综合实战      │
│ 基础          │    │ 进阶          │    │ 配置管理      │    │ IaC 全流程   │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：Terraform 基础 — 用代码管理云资源

**目标：理解声明式基础设施的核心概念，能用 Terraform 创建和管理云资源**

#### Day 1：为什么需要 IaC

| 主题 | 内容 |
|------|------|
| 手动操作的痛点 | 点鼠标不可复制、容易出错、改完就忘 |
| ClickOps vs GitOps | 手动点 UI vs Git 里声明状态，工具收敛 |
| IaC 的核心价值 | 可复现、可版本管理、可 Code Review、可自动化 |
| 声明式 vs 命令式 | Terraform（声明式，描述我想要什么）vs Ansible（命令式，告诉我该怎么做） |
| IaC 工具全景 | Terraform / OpenTofu / Pulumi / Ansible / Chef / Salt / CloudFormation |

**练习：** 创建一个空目录，初始化 Terraform 项目

```bash
mkdir terraform-learn && cd terraform-learn
```

#### Day 2：Terraform 核心工作流

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1. 编写配置 | `main.tf` | 声明期望的资源状态 |
| 2. 初始化 | `terraform init` | 下载 provider 插件、初始化 backend |
| 3. 计划 | `terraform plan` | 对比当前状态和期望状态，生成执行计划 |
| 4. 应用 | `terraform apply` | 执行计划，创建/修改/删除资源 |
| 5. 销毁 | `terraform destroy` | 删除所有 Terraform 管理的资源 |

**练习：** 安装 Terraform，跑通 `terraform version`

#### Day 3：HCL 语法基础

| 概念 | 格式 |
|------|------|
| `resource` | `resource "type" "name" { ... }` — 声明一个资源 |
| `variable` | `variable "name" { type = ... default = ... }` — 输入变量 |
| `output` | `output "name" { value = ... }` — 导出值供外部使用 |
| `locals` | `locals { name = expression }` — 局部值 |
| `data` | `data "type" "name" { ... }` — 读取已存在的资源 |
| 基本类型 | `string` / `number` / `bool` / `list(...)` / `map(...)` |

**练习：**
```hcl
# main.tf
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

variable "container_name" {
  type    = string
  default = "terraform-nginx"
}

resource "docker_image" "nginx" {
  name = "nginx:latest"
}

resource "docker_container" "nginx" {
  name  = var.container_name
  image = docker_image.nginx.image_id
  ports {
    internal = 80
    external = 8080
  }
}

output "container_id" {
  value = docker_container.nginx.id
}
```

#### Day 4：State — Terraform 的核心

| 主题 | 内容 |
|------|------|
| State 文件 | `terraform.tfstate` — JSON 文件，记录 Terraform 管理的资源与真实资源的映射 |
| 为什么 State 重要 | plan 需要对比当前 state 和期望配置的差异 |
| 本地 State 的问题 | 多人协作时会冲突；无法分享 |
| Remote Backend | 把 state 存在远程（S3 / GCS / Terraform Cloud）|
| State Locking | 防止两个人同时 apply |

**练习：**
```bash
# 观察 state
terraform apply
cat terraform.tfstate                   # 看看里面的 JSON 结构
terraform state list                    # 列出所有被管理的资源
terraform state show docker_container.nginx  # 查看单个资源详情
```

#### Day 5：依赖图与资源引用

| 主题 | 内容 |
|------|------|
| 隐式依赖 | 通过引用 `<type>.<name>.<attribute>` 自动推断 |
| 显式依赖 | `depends_on` — 强制指定依赖关系 |
| 有向无环图 | Terraform 根据依赖关系生成 DAG，决定创建/删除顺序 |
| `terraform graph` | 生成依赖图（转 Graphviz） |

**练习：**
```bash
terraform graph | dot -Tpng > graph.png  # 可视化依赖关系（需安装 graphviz）
```

观察 Terraform 如何根据 `docker_image.nginx.image_id` 的引用，自动判断先拉镜像再创建容器

#### Day 6：变量、输出和 terraform.tfvars

| 主题 | 内容 |
|------|------|
| 变量定义 | `variable` 块，可设 `type` / `default` / `validation` / `sensitive` |
| 赋值方式 | CLI `-var`、`-var-file`、环境变量 `TF_VAR_name`、`.auto.tfvars` |
| 优先级 | CLI > `*.auto.tfvars` > `terraform.tfvars` > 环境变量 > default |
| 输出 | `output` 块，apply 后打印，`terraform output` 查看 |

**练习：**
```hcl
variable "environment" {
  type    = string
  default = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod"
  }
}

# terraform.tfvars
environment = "staging"
```

#### Day 7：第一周综合练习

```
目标：用 Terraform 在本地 Docker 环境中管理完整的应用栈

任务：
1. 用 Terraform 的 Docker provider 管理以下资源：
   ├── nginx 容器（端口 8080）
   ├── Python Web 容器（自己写的，端口 5000）
   └── PostgreSQL 容器（端口 5432，带 volume 持久化）

2. 使用变量：
   - 所有端口号可配置
   - postgres 密码为 sensitive 变量
   - 区分 dev/prod 环境的 tfvars 文件

3. 使用 output 导出：
   - nginx 容器名
   - postgres 容器 IP（通过 docker network）

4. 验证：
   - terraform plan 输出清晰可读
   - terraform apply 一次性创建所有资源
   - terraform destroy 一次性清理所有资源
   - 修改端口号后再次 apply，观察哪些资源需要重建
```

---

### 第2周：Terraform 进阶

**目标：掌握 module、workspace、循环、条件等生产级用法**

#### Day 8：Module — 可复用的基础设施模块

| 主题 | 内容 |
|------|------|
| 为什么需要 Module | 避免复制粘贴；把一组相关资源打包为可复用单元 |
| Module 结构 | `main.tf` / `variables.tf` / `outputs.tf` / `README.md` |
| 调用方式 | `module "name" { source = "./path" param = "value" }` |
| 常用 source | 本地路径 / Git / Terraform Registry |
| Module 版本管理 | Git tag / Registry version |

**练习：** 将第1周的 nginx 容器配置提取为 Module，让 `container_name` 和 `port` 参数化

```hcl
# modules/nginx/main.tf
resource "docker_container" "nginx" {
  name  = var.name
  image = "nginx:${var.nginx_version}"
  ports {
    internal = 80
    external = var.external_port
  }
}

# 使用
module "web_nginx" {
  source        = "./modules/nginx"
  name          = "web-nginx"
  external_port = 8080
}

module "api_nginx" {
  source        = "./modules/nginx"
  name          = "api-nginx"
  external_port = 8081
}
```

#### Day 9：循环 — for_each & count

| 方式 | 说明 |
|------|------|
| `count` | 创建 N 个相同资源，用索引区分：`count` + `count.index` |
| `for_each` | 遍历 map/set，每个元素一个实例（确定性 key，增删元素不影响其他） |
| 原则 | **优先用 for_each**，因为 key 确定，增删中间元素不影响其他资源 |
| `for` 表达式 | 列表/映射转换：`[for s in list : upper(s)]` / `{for k, v in map : k => upper(v)}` |

**练习：**
```hcl
# for_each 创建多个用户
variable "users" {
  type = map(object({
    name = string
    role = string
  }))
}

resource "some_user" "users" {
  for_each = var.users
  username = each.value.name
  role     = each.value.role
}

# dynamic block --- 动态嵌套块
resource "docker_container" "app" {
  name  = "app"
  image = "app:latest"
  dynamic "ports" {
    for_each = var.ports
    content {
      internal = ports.value.internal
      external = ports.value.external
    }
  }
}
```

#### Day 10：条件表达式与函数

| 用法 | 示例 |
|------|------|
| 三元表达式 | `var.env == "prod" ? 3 : 1` |
| `count` 条件创建 | `count = var.create_backup ? 1 : 0` |
| `try()` | `try(var.optional_field, "default")` |
| `lookup()` | `lookup(map, key, default)` |
| `coalesce()` | `coalesce(var.a, var.b, "fallback")` |
| `can()` | `can(var.list[0])` 判断是否可行不报错 |
| `one()` | `one([for x in list : x if x.enabled])` 取唯一匹配 |

**练习：** 在 module 中使用条件语句：prod 环境创建多副本 + 大规格；dev 环境只创建 1 个

#### Day 11：for_each & count 的正确选择

| 场景 | 选择 | 原因 |
|------|------|------|
| 创建 N 个相同的网卡 | `count` | 索引无所谓 |
| 按用户列表创建账号 | `for_each` | 用用户名做 key，删一个人不影响其他人 |
| 条件创建单个资源 | `count = condition ? 1 : 0` | 简单场景 |
| 按配置 map 动态创建资源 | `for_each` | key 明确，确定性强 |

**练习：** 用 `for_each` 管理多个 Docker 容器，尝试在 map 中增删一项，观察 plan 输出（只有增删的项受影响）

#### Day 12：Remote State & Backend

| 主题 | 内容 |
|------|------|
| Remote Backend 类型 | S3 / GCS / Azure Storage / PostgreSQL / Terraform Cloud |
| Backend 配置 | `terraform { backend "s3" { ... } }` |
| State Locking | S3 用 DynamoDB 做锁；GCS 原生支持 |
| Backend 初始化 | `terraform init` 会迁移本地 state 到 remote |

**练习：** 用本地文件模拟 remote state（实际项目中用 S3/GCS）：

```hcl
terraform {
  backend "local" {
    path = "/tmp/terraform-backend/state.tfstate"
  }
}
# 体验 backend 迁移：init → 选择 yes → state 从本地移到新位置
```

#### Day 13：Workspace — 多环境管理

| 主题 | 内容 |
|------|------|
| Workspace 是什么 | 同一套配置的多个 state 实例 |
| 默认 workspace | `default` |
| 创建 workspace | `terraform workspace new staging` |
| 切换 | `terraform workspace select staging` |
| `terraform.workspace` | 在配置中获取当前 workspace 名 |
| Workspace vs 目录分离 | Workspace 适合环境参数差异小；目录适合差异大 |

**练习：** 为同一套 Docker 配置创建 `dev` / `staging` / `prod` 三个 workspace，每个使用不同的端口和副本数

#### Day 14：第二周综合练习

```
目标：构建一个参数化、可复用、多环境的 Terraform 项目

项目结构：
terraform-docker-app/
├── main.tf                    # 调用 modules
├── variables.tf               # 全局变量
├── outputs.tf                 # 全局输出
├── terraform.tfvars           # 默认值
├── envs/
│   ├── dev.tfvars
│   └── prod.tfvars
└── modules/
    ├── network/               # Docker 网络 module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── web/                   # Web 应用 module
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── database/              # 数据库 module
        ├── main.tf
        ├── variables.tf
        └── outputs.tf

要求：
1. 每个 Module 都有 version 参数
2. 使用 for_each 管理多实例部署（dev 1 实例，prod 3 实例）
3. 使用 workspace 区分 dev / prod
4. 使用条件表达式（prod 用大规格，dev 用小规格）
5. terraform plan 针对 dev/prod 分别输出正确的变更计划
```

---

### 第3周：Ansible — 配置管理与自动化

**目标：理解命令式配置管理，能用 Ansible 管理服务器配置**

#### Day 15：Ansible 是什么

| 主题 | 内容 |
|------|------|
| 命令式 vs 声明式 | Ansible（命令式：一步步执行） vs Terraform（声明式：描述最终状态） |
| Ansible 核心优势 | 无 agent（SSH 连接）、YAML 语法、模块丰富 |
| 适用场景 | 配置管理、应用部署、批量操作、编排 |
| Terraform + Ansible 分工 | Terraform 管基础设施（虚拟机/网络）；Ansible 管机器上的软件和配置 |

**练习：** 安装 Ansible，跑通首个命令

```bash
pip install ansible
ansible --version
ansible localhost -m ping               # 对本机执行 ping 模块
```

#### Day 16：Inventory — 主机清单

| 格式 | 示例 |
|------|------|
| INI | `[webservers]` / `web01 ansible_host=10.0.0.1` |
| YAML | `all: hosts: web01: ansible_host: 10.0.0.1` |
| 分组 | `[webservers:children]` — 组的组 |
| 动态 Inventory | 从云 API 自动生成（AWS / GCP plugin） |
| `ansible-inventory` | 查看解析后的 inventory |

**练习：**
```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        web-01:
          ansible_host: localhost
          ansible_connection: local
    databases:
      hosts:
        db-01:
          ansible_host: localhost
          ansible_connection: local
```

#### Day 17：Ad-Hoc 命令与模块

| 模块 | 用途 |
|------|------|
| `ansible.builtin.command` | 执行任意命令（默认模块） |
| `ansible.builtin.shell` | 执行 shell（支持管道、重定向） |
| `ansible.builtin.copy` | 复制文件到远程 |
| `ansible.builtin.file` | 管理文件/目录权限 |
| `ansible.builtin.apt/yum/pip` | 包管理 |
| `ansible.builtin.service` | 管理 systemd 服务 |
| `ansible.builtin.template` | 渲染 Jinja2 模板到远程 |

**练习：**
```bash
ansible all -m copy -a "src=/tmp/test.txt dest=/tmp/test.txt"
ansible webservers -m apt -a "name=nginx state=present" --become
ansible all -m service -a "name=nginx state=started enabled=yes" --become
ansible all -m setup                    # 收集所有主机信息（facts）
```

#### Day 18：Playbook — 剧本编排

| 概念 | 说明 |
|------|------|
| Playbook | YAML 文件，定义一组 Play |
| Play | 一个 Play = 在哪些主机上执行哪些 Task |
| Task | 一个 Task = 调用一个模块并传参 |
| Handler | 被 Task notify 触发执行（如：配置文件变了 → 重启服务） |
| `--check` | 干跑模式（Dry Run），只看不动 |
| `--diff` | 显示变更内容 |

**练习：** 写你的第一个 Playbook

```yaml
# playbook.yml
- name: Configure web server
  hosts: webservers
  become: yes
  vars:
    nginx_port: 80

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Copy index.html
      template:
        src: index.html.j2
        dest: /var/www/html/index.html
      notify: reload nginx

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

#### Day 19：Jinja2 模板与 Facts

| 主题 | 内容 |
|------|------|
| Facts | Ansible 自动收集的主机信息（OS、IP、CPU、内存...） |
| `ansible_facts` | 在模板中引用事实 |
| Jinja2 模板 | 用 `{{ variable }}` 动态渲染配置文件 |
| 条件 | `when: ansible_facts['os_family'] == "Debian"` |
| 循环 | `loop:` / `with_items:` |

**练习：** 用 Jinja2 模板生成 nginx 配置，根据 fact 自动适配：Debian 用 `sites-available`，RHEL 用 `conf.d`

#### Day 20：Roles — Ansible 的 Module

| 主题 | 内容 |
|------|------|
| Role 目录结构 | `tasks/` / `handlers/` / `templates/` / `vars/` / `defaults/` / `meta/` |
| 创建 Role | `ansible-galaxy init nginx_role` |
| 使用 Role | 在 Playbook 中 `roles:` 列表引用 |
| Ansible Galaxy | 社区 Role 市场 |
| 变量优先级 | extra vars > play vars > role vars > role defaults |

**练习：** 将 Day 18 的 nginx Playbook 重构为一个标准的 Ansible Role

```bash
ansible-galaxy init roles/nginx
# 将 tasks / templates / handlers 分别放到对应目录
# 在 playbook 中引用这个 role
```

#### Day 21：第三周综合练习

```
目标：用 Ansible 配置一台"学习用 Linux 服务器"

Playbook 应完成以下配置：

1. 基础配置：
   - 创建学习用户（免密 sudo）
   - 配置 SSH（禁用 root 登录、禁用密码认证）
   - 配置时区和 locale

2. 开发环境：
   - 安装 Python 3.12 + pip + venv
   - 安装 Docker（官方仓库版，含 docker-compose）
   - 安装常用工具：git / vim / tmux / htop / jq

3. 配置 zsh + oh-my-zsh（通过 Role）：
   - 使用 ansible-galaxy 安装社区 zsh role
   - 或自己写 role

4. 部署监控 agent：
   - 用 template 模块生成 node_exporter systemd unit
   - 启动并 enable node_exporter

项目结构：
ansible-server-setup/
├── ansible.cfg                 # 配置（关闭 host_key_checking 等）
├── inventory.yml               # 主机清单
├── site.yml                    # 主 Playbook
├── group_vars/
│   └── all.yml                 # 全局变量
├── roles/
│   ├── common/                 # 基础配置 role
│   ├── devtools/               # 开发工具 role
│   └── monitoring/             # 监控 agent role
└── requirements.yml            # galaxy 依赖
```

---

### 第4周：综合实战 — IaC 全流程

**目标：串联 Terraform + Ansible + CI/CD，实现完整的 IaC 流水线**

#### Day 22：Terraform 创建基础设施 + Ansible 配置

| 主题 | 内容 |
|------|------|
| Terraform 调用 Ansible | `local-exec` provisioner 在资源创建后执行 ansible-playbook |
| 更好的方式 | CI/CD Pipeline 中先 `terraform apply`，再 `ansible-playbook` |
| Terraform output → Ansible | `terraform output -json` 生成动态 inventory |

**练习：** 编写一个脚本，从 Terraform output 生成 Ansible inventory

```bash
#!/bin/bash
# generate_inventory.sh
terraform -chdir=infra output -json web_ips | \
  jq -r 'to_entries | map("\(.key) ansible_host=\(.value)") | .[]' \
  > ansible/inventory/hosts
```

#### Day 23：GitHub Actions 中运行 Terraform

| 主题 | 内容 |
|------|------|
| Terraform in CI | PR 时自动 `terraform plan`，merge 时自动 `terraform apply` |
| 认证 | 云平台凭据存在 GitHub Secrets |
| State | 使用 Remote Backend（S3/GCS）+ Locking |
| Plan 评论 | 将 plan 结果贴在 PR 评论中 |

**练习：** 编写 GitHub Actions Workflow

```yaml
name: Terraform
on:
  pull_request:
    paths: [ 'terraform/**' ]
  push:
    branches: [ main ]
    paths: [ 'terraform/**' ]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
        working-directory: terraform
      - run: terraform fmt -check -recursive
        working-directory: terraform
      - run: terraform plan -out=tfplan
        working-directory: terraform
        if: github.event_name == 'pull_request'
      - run: terraform apply tfplan
        working-directory: terraform
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

#### Day 24：GitHub Actions 中运行 Ansible

| 主题 | 内容 |
|------|------|
| SSH 认证 | GitHub Secrets 存 SSH 私钥 |
| `ansible-lint` | 检查 Playbook 质量 |
| Dry Run | PR 时 `--check --diff`，merge 时正式 apply |
| Rollback | Playbook 保守设计：强调幂等性，确保重跑安全 |

**练习：** 编写 Ansible CI Workflow

```yaml
name: Ansible
on:
  pull_request:
    paths: [ 'ansible/**' ]
  push:
    branches: [ main ]
    paths: [ 'ansible/**' ]

jobs:
  ansible:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ansible ansible-lint
      - run: ansible-lint ansible/site.yml
      - run: ansible-playbook ansible/site.yml --check --diff
        if: github.event_name == 'pull_request'
      - run: ansible-playbook ansible/site.yml
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

#### Day 25：Terraform Provisioner 与 Ansible 的协作模式

| 模式 | 说明 | 推荐度 |
|------|------|--------|
| Terraform `local-exec` 调 Ansible | 在 apply 后自动执行 Playbook | 适合小项目 |
| CI Pipeline 串联 | CI 中先 TF 后 Ansible | 生产级推荐 |
| Ansible 调 Terraform | Ansible 的 `terraform` 模块管理 TF | 少用，角色混淆 |
| Packer + Ansible + Terraform | Ansible 构建镜像，Terraform 部署 | 高级模式 |

**练习：** 实现 CI Pipeline 串联模式：`terraform apply` → 生成 inventory → `ansible-playbook`

#### Day 26：K8s 的 IaC — Helm 与 Kustomize

| 主题 | 内容 |
|------|------|
| Helm | K8s 的包管理器 — 把一组 YAML 打包为 Chart + values 参数化 |
| Values 分层 | `values.yaml`（默认）→ `values-{env}.yaml`（覆盖）→ `--set`（CLI 覆盖） |
| Terraform Helm Provider | 用 Terraform 管理 Helm Release |
| Kustomize | 无模板的 K8s 配置定制（修改已有 YAML 的特定字段） |
| Terraform Kustomize Provider | 用 Terraform 管理 Kustomize overlay |

**练习：** 用 Terraform 的 Helm Provider 部署 nginx-ingress 到 K8s

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  namespace  = "ingress-nginx"
  create_namespace = true

  set {
    name  = "controller.replicaCount"
    value = var.environment == "prod" ? 3 : 1
  }
}
```

#### Day 27：可观测性服务的 IaC 化

| 主题 | 内容 |
|------|------|
| 设计 IaC 项目结构 | `infra/` (Terraform) + `config/` (Ansible) + `apps/` (Helm/Kustomize) |
| IaC 仓库与 App 仓库分离 | App 代码和 Infra 代码独立演进 |

**练习：** 将之前学习过的监控栈（Prometheus + Grafana + Loki + Jaeger）全部 IaC 化

#### Day 28：第四周综合练习

```
目标：实现一个完整的 IaC 项目

项目：在本地 Docker/K8s 环境中，用 IaC 工具链管理一切

阶段一：基础设施（Terraform）
├── Docker 网络
├── PostgreSQL 数据库
└── K8s 集群（kind 或 minikube）

阶段二：中间件部署（Terraform + Helm）
├── Helm Release: Prometheus Stack
├── Helm Release: Loki Stack
├── Helm Release: Jaeger
└── Helm Release: nginx-ingress

阶段三：应用部署（Terraform + Helm）
├── Python Web 应用（自写的）
├── Python Worker 应用（自写的）
└── 对应的 ServiceMonitor

阶段四：主机配置（Ansible）
├── 基础安全配置
├── Docker 安装与配置
└── 监控 agent（node_exporter + Grafana Agent）

阶段五：CI/CD
├── GitHub Actions Workflow: 自动 terraform plan（PR）
├── GitHub Actions Workflow: 自动 terraform apply（merge）
└── GitHub Actions Workflow: 自动 ansible-playbook（merge）

产出物：
1. IaC 仓库：完整的 Terraform + Ansible + Helm 代码
2. CI/CD 配置：GitHub Actions Workflow 文件
3. README：如何初始化、如何部署、如何销毁
4. 架构图：资源拓扑 + 数据流向
```

---

## 四、里程碑检查点

```
Week 1 结束：✓ 能解释声明式 IaC 的核心思想
             ✓ 能用 Terraform 管理 Docker 容器的完整生命周期
             ✓ 理解 State 的作用和 Remote Backend 的必要性
Week 2 结束：✓ 能编写可参数化的 Module，用 for_each 管理同类资源
             ✓ 能用 workspace 管理多环境
             ✓ 能组织一个多 module、多环境的 Terraform 项目
Week 3 结束：✓ 能编写 Ansible Playbook 和 Role
             ✓ 能用 Jinja2 + Facts 生成动态配置
             ✓ 知道 Terraform 和 Ansible 的分工边界
Week 4 结束：✓ 能在 CI/CD 中集成 Terraform 和 Ansible
             ✓ 能写出生产可用的 IaC 项目结构
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 官方文档 | Terraform 官方文档 — developer.hashicorp.com/terraform |
| 官方文档 | Ansible 官方文档 — docs.ansible.com |
| 书籍 | 《Terraform: Up & Running》（3rd edition） |
| 书籍 | 《Ansible for DevOps》— Jeff Geerling |
| 在线练习 | Terraform 官方教程 — developer.hashicorp.com/terraform/tutorials |
| 工具 | `tflint`（Terraform 静态检查） |
| 工具 | `checkov`（IaC 安全扫描） |
| 工具 | `tfsec`（Terraform 安全扫描） |
| 工具 | `ansible-lint`（Ansible Playbook 质量检查） |
| 工具 | `terraform-docs`（自动生成 Module 文档） |
| 视频 | "Complete Terraform Course" — freeCodeCamp（YouTube） |
| 社区 | Terraform Registry — registry.terraform.io |
