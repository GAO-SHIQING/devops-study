# Study

个人技术学习仓库，涵盖运维/DevOps/SRE 方向的系统化学习笔记。

## 目录结构

```
study/
├── README.md                    # 本文件
├── *_learning_plan.md           # 各主题学习路线规划
├── *_notebooks/                 # 各主题 Jupyter 笔记
├── docs/                        # 项目文档
└── test/                        # 测试脚本
```

## 学习主题

| 主题 | 学习计划 | 笔记 | 说明 |
|------|---------|------|------|
| **Python** | [python_learning_plan.md](./python_learning_plan.md) | [python_notebooks/](./python_notebooks/) | 零基础到高级特性，含 OOP、文件 IO 等 |
| **Docker** | [docker_learning_plan.md](./docker_learning_plan.md) | [docker_notebooks/](./docker_notebooks/) | 容器基础、Dockerfile、Compose、进阶实战 |
| **Kubernetes** | [k8s_learning_plan.md](./k8s_learning_plan.md) | [k8s_notebooks/](./k8s_notebooks/) | 核心概念、网络存储、调度运维、生产实践 |
| **Linux** | [linux_learning_plan.md](./linux_learning_plan.md) | [linux_notebooks/](./linux_notebooks/) | 用户空间、systemd、网络、性能排障 |
| **SQL** | [sql_learning_plan.md](./sql_learning_plan.md) | [sql_notebooks/](./sql_notebooks/) | CRUD、查询、表设计、Python 集成 |
| **CI/CD** | [cicd_learning_plan.md](./cicd_learning_plan.md) | [cicd_notebooks/](./cicd_notebooks/) | 基础概念、构建测试、Docker 部署、进阶 |
| **Git** | [git_learning_plan.md](./git_learning_plan.md) | [git_notebooks/](./git_notebooks/) | 内部原理、分支合并、进阶技巧、协作工作流 |
| **监控可观测** | [monitoring_learning_plan.md](./monitoring_learning_plan.md) | [monitoring_notebooks/](./monitoring_notebooks/) | Prometheus、Grafana、Loki、OpenTelemetry |
| **IaC** | [iac_learning_plan.md](./iac_learning_plan.md) | [iac_notebooks/](./iac_notebooks/) | Terraform、Ansible、Helm、IaC 全流程 |

## 学习路线建议

```
Python 基础 → Docker → Kubernetes → Linux 深入 → SQL → CI/CD → Git 深入 → 监控可观测 → IaC
```

1. **Python** — 先掌握一门编程语言，后续自动化、数据处理都依赖
2. **Docker** — 理解容器化，是 K8s 和 CI/CD 的前置知识
3. **Kubernetes** — 容器编排，生产环境核心技能
4. **Linux** — 深入系统底层，排障和性能优化必备
5. **SQL** — 数据库操作，后端开发基础
6. **CI/CD** — 自动化流水线，串联前面所有技能
7. **Git** — 深入版本控制的内部原理和团队协作工作流
8. **监控可观测** — Prometheus + Grafana + 日志 + 链路追踪，SRE 核心能力
9. **IaC** — Terraform + Ansible，用代码管理一切基础设施

## 使用方式

每个主题的 `.md` 文件是学习路线规划，`.ipynb` 文件是配套的 Jupyter Notebook 笔记。建议按顺序阅读规划文档，再打开对应 notebook 动手实践。

```bash
# 启动 Jupyter
jupyter notebook
```
