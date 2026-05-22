# Kubernetes 零基础学习路线

---

## 一、前置条件

- 已掌握 Docker（容器、镜像、Dockerfile、Compose）
- 理解基本的网络概念（IP、端口、DNS）
- 有一个可用的 Linux 环境（或 macOS）

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ 核心概念      │    │ 网络与存储    │    │ 调度与管理    │    │ 实战综合      │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：核心概念 — 理解 K8s 的"世界观"

**目标：搭建环境，理解 Pod、Deployment、Service 三个核心对象**

#### Day 1：为什么需要 K8s

| 主题 | 内容 |
|------|------|
| 从单机到集群 | 单机部署 → 多机手动管理 → 容器编排 |
| K8s 做什么 | 调度、自愈、伸缩、服务发现、滚动更新 |
| 架构概览 | 控制平面（API Server / Scheduler / Controller Manager / etcd）+ 工作节点（kubelet / kube-proxy） |

**练习：** 安装 minikube 或 kind，跑通 `kubectl cluster-info`

#### Day 2：kubectl 初体验

| 主题 | 命令 |
|------|------|
| 集群信息 | `kubectl cluster-info` / `kubectl version` |
| 节点 | `kubectl get nodes` / `kubectl describe node` |
| 命名空间 | `kubectl get ns` / `kubectl create ns` |
| 上下文 | `kubectl config get-contexts` / `kubectl config use-context` |

**练习：** 探索 minikube 集群，查看所有节点和系统命名空间

#### Day 3：Pod — 最小调度单元

| 主题 | 说明 |
|------|------|
| Pod 是什么 | 一个或多个容器的组合，共享网络和存储 |
| 声明式 vs 命令式 | `kubectl run` vs YAML 文件 |
| Pod 生命周期 | Pending → Running → Succeeded/Failed |
| 查看与调试 | `kubectl get pods` / `kubectl describe pod` / `kubectl logs` |

**练习：**
```yaml
# 创建第一个 Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
spec:
  containers:
  - name: hello
    image: python:3.12-slim
    command: ["python", "-c", "print('Hello K8s!')"]
```
```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl logs my-first-pod
```

#### Day 4：Deployment — 声明式部署

| 主题 | 说明 |
|------|------|
| 为什么需要 Deployment | 管理 Pod 副本、滚动更新、回滚 |
| ReplicaSet | Deployment 自动管理 RS，RS 管理 Pod |
| 核心字段 | replicas / selector / template |
| 常用命令 | `kubectl create deploy` / `kubectl scale` / `kubectl rollout` |

**练习：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: python-api
  template:
    metadata:
      labels:
        app: python-api
    spec:
      containers:
      - name: api
        image: python:3.12-slim
        command: ["python", "-c", "import http.server; http.server.HTTPServer(('', 8000), http.server.SimpleHTTPRequestHandler).serve_forever()"]
        ports:
        - containerPort: 8000
```
体验：扩缩容 `kubectl scale`、滚动更新 `kubectl set image`、回滚 `kubectl rollout undo`

#### Day 5：Service — 网络暴露

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| ClusterIP | 集群内部访问 | 服务间通信 |
| NodePort | 节点端口暴露 | 开发调试 |
| LoadBalancer | 云负载均衡器 | 生产对外 |
| ExternalName | DNS 别名 | 外部服务映射 |

**练习：** 创建 Service 暴露前一天的 Deployment，体会三种类型的区别

#### Day 6：ConfigMap & Secret

| 主题 | 说明 |
|------|------|
| ConfigMap | 非敏感配置：环境变量、配置文件 |
| Secret | 敏感信息：密码、Token、证书 |
| 挂载方式 | 环境变量 / 文件挂载 |

**练习：** 将 Python 应用的数据库连接串、API Key 分别放 ConfigMap 和 Secret 中

#### Day 7：第一周综合练习

把 Docker 阶段写的 Flask + Redis + DB 应用迁移到 K8s：

```
创建以下资源：
├── Namespace: my-app
├── Deployment: web (3 副本)
├── Deployment: redis (1 副本)
├── Deployment: postgres (1 副本)
├── ConfigMap: 应用配置
├── Secret: 数据库密码
├── Service: web (ClusterIP)
├── Service: redis (ClusterIP)
└── Service: postgres (ClusterIP)
```

---

### 第2周：网络与存储

**目标：理解 K8s 网络模型，能管理持久化数据**

#### Day 8：K8s 网络模型

| 主题 | 说明 |
|------|------|
| 核心原则 | 所有 Pod 可以互相通信（无 NAT） |
| CNI 插件 | Calico / Flannel / Cilium |
| Pod 网络 | 每个 Pod 有独立 IP |
| Service 网络 | kube-proxy 实现负载均衡 |

#### Day 9：Ingress — HTTP 路由

| 主题 | 说明 |
|------|------|
| Ingress 是什么 | 七层负载均衡，基于域名/路径路由 |
| Ingress Controller | nginx-ingress / traefik |
| 配置示例 | 多域名、TLS 证书、路径重写 |

**练习：**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: api.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 8000
```

#### Day 10：存储基础

| 概念 | 说明 |
|------|------|
| Volume | Pod 级别的临时存储，随 Pod 生命周期 |
| PersistentVolume (PV) | 集群级别的存储资源 |
| PersistentVolumeClaim (PVC) | 用户对存储的申请 |
| StorageClass | 动态 PV 供给 |

**练习：** 创建 PVC，挂载到 Pod 中验证数据持久化

#### Day 11：StatefulSet

| 对比 Deployment | StatefulSet |
|-----------------|-------------|
| Pod 标识 | 随机名称 | 有序名称（app-0, app-1, ...） |
| 网络标识 | 无稳定标识 | 稳定的 DNS 名称 |
| 存储 | 共享 PVC | 每个 Pod 独立 PVC |
| 适用场景 | 无状态应用 | 数据库、消息队列 |

**练习：** 用 StatefulSet 部署一个 Redis 集群

#### Day 12：Service 深入

| 主题 | 说明 |
|------|------|
| Headless Service | `clusterIP: None`，直接返回 Pod IP |
| Endpoint / EndpointSlice | Service 后端 Pod 的 IP 列表 |
| ExternalName | 将外部服务映射到集群 DNS |
| kube-proxy 模式 | iptables / IPVS |

#### Day 13：网络策略（NetworkPolicy）

| 主题 | 说明 |
|------|------|
| 默认策略 | 默认所有 Pod 互通 |
| 入站规则 | 控制谁可以访问我 |
| 出站规则 | 控制我可以访问谁 |
| namespaceSelector + podSelector | 精细控制 |

**练习：** 配置 NetworkPolicy，限制数据库 Pod 只允许 Web Pod 访问

#### Day 14：第二周综合练习

为应用做好网络和存储规划：

```
要求：
├── Ingress 配置两个域名：api.myapp.local / admin.myapp.local
├── Web 服务使用 PVC 存储上传文件
├── 数据库使用 StatefulSet + PVC 确保数据不丢
├── NetworkPolicy：DB 只允许 Web 和 Worker 访问
└── 所有敏感信息在 Secret 中
```

---

### 第3周：调度、管理与运维

**目标：理解调度机制，能做日常运维**

#### Day 15：调度机制

| 主题 | 说明 |
|------|------|
| 调度流程 | Filter → Score → Bind |
| nodeSelector | 简单的节点选择 |
| 亲和性（affinity） | nodeAffinity / podAffinity / podAntiAffinity |
| 污点与容忍（Taint/Toleration） | 排斥/允许 Pod 调度到特定节点 |

**练习：** 配置 podAntiAffinity，将 Web 的多个副本分散到不同节点

#### Day 16：资源管理

| 主题 | 说明 |
|------|------|
| requests | 调度时预留的资源（决定调度到哪个节点） |
| limits | 运行时上限（超了会被 throttle 或 OOM Kill） |
| QoS 等级 | Guaranteed / Burstable / BestEffort |
| LimitRange | 命名空间级别的默认资源限制 |
| ResourceQuota | 命名空间级别的资源配额 |

**练习：** 为所有服务配置合理的 requests/limits，观察 OOM 行为

#### Day 17：健康检查

| 探针类型 | 说明 |
|----------|------|
| livenessProbe | 容器是否活着？失败 → 重启 |
| readinessProbe | 容器是否就绪？失败 → 移出 Service |
| startupProbe | 启动是否完成？用于慢启动容器 |

**练习：** 为 Python Web 应用添加 liveness 和 readiness 探针，测试失败场景

#### Day 18：HPA — 水平自动伸缩

| 主题 | 说明 |
|------|------|
| HPA 原理 | 根据 CPU/内存/自定义指标自动调整副本数 |
| metrics-server | 资源指标采集 |
| 配置示例 | minReplicas / maxReplicas / 触发阈值 |

**练习：** 给 Web 服务配置 HPA，用压测工具触发自动伸缩

#### Day 19：Helm 入门

| 主题 | 说明 |
|------|------|
| Helm 是什么 | K8s 的包管理器 |
| Chart 结构 | Chart.yaml / values.yaml / templates/ |
| 常用命令 | `helm install` / `helm upgrade` / `helm rollback` / `helm repo` |
| 模板语法 | `{{ .Values.xxx }}`、`{{ if }}`、`{{ range }}` |

**练习：** 将自己写的一堆 YAML 文件改造成一个 Helm Chart

#### Day 20：RBAC — 权限控制

| 概念 | 说明 |
|------|------|
| ServiceAccount | Pod 使用的身份 |
| Role / ClusterRole | 定义权限集合 |
| RoleBinding / ClusterRoleBinding | 绑定身份和权限 |

**练习：** 创建一个只读权限的 ServiceAccount，给监控工具使用

#### Day 21：第三周综合练习

```
对应用进行全面加固：
├── 所有服务配置 requests/limits
├── Web 服务配置 HPA（min 2, max 10）
├── 添加 liveness + readiness 探针
├── 用 Helm 打包整个应用
├── RBAC 限制各服务的权限
└── 压测验证：自动伸缩是否生效
```

---

### 第4周：生产实践

#### Day 22：日志管理

| 主题 | 说明 |
|------|------|
| 日志架构 | 应用 → stdout → 容器运行时 → 日志收集 |
| kubectl logs | 单 Pod 日志查看 |
| 日志收集方案 | EFK（Elasticsearch + Fluentd + Kibana）/ Loki + Grafana |
| 结构化日志 | JSON 格式日志，方便检索 |

**练习：** 让 Python 应用输出 JSON 格式日志，配置 Fluentd 收集

#### Day 23：监控与告警

| 主题 | 说明 |
|------|------|
| Prometheus | 指标采集与时序数据库 |
| Grafana | 可视化面板 |
| 关键指标 | Pod 状态、CPU/内存使用率、请求延迟、错误率 |
| AlertManager | 告警规则与通知 |

**练习：** 部署 Prometheus + Grafana，为应用配置基础监控面板

#### Day 24：排障实战

| 场景 | 排查思路 |
|------|----------|
| Pod 起不来 | `describe pod` → Events / `logs` → 镜像拉取 / 资源不够 / 健康检查失败 |
| 服务访问不通 | Service 是否存在 → Endpoint 是否有 Pod → NetworkPolicy 是否拦截 |
| 应用报错 | `kubectl exec` 进容器 → 查日志 → 查环境变量 → 查配置挂载 |
| 节点异常 | `describe node` → Conditions → 资源压力 |

**练习：** 人为制造故障（错误镜像、端口写错、资源不足），逐个排查修复

#### Day 25：CI/CD 集成

| 主题 | 说明 |
|------|------|
| GitOps 概念 | Git 作为单一事实来源 |
| ArgoCD / Flux | GitOps 工具 |
| 镜像更新策略 | 自动更新 Deployment 镜像 |
| 部署策略 | 滚动更新 / 蓝绿部署 / 金丝雀发布 |

**练习：** 用 GitHub Actions 实现：push 代码 → 构建镜像 → 更新 K8s 部署

#### Day 26：安全最佳实践

| 实践 | 说明 |
|------|------|
| Pod Security Standards | privileged / baseline / restricted |
| 镜像扫描 | Trivy 集成到 CI |
| 网络策略 | 最小权限，禁止不必要的互通 |
| Secret 加密 | 不要只用 base64，考虑 Sealed Secrets 或 External Secrets |
| 只读根文件系统 | `readOnlyRootFilesystem: true` |

#### Day 27-28：第四周综合项目

从零部署一个生产级 Python 应用：

```
项目：博客系统 K8s 部署

要求：
├── 前端 Nginx（2 副本，HPA）
├── API 服务 FastAPI（3 副本，HPA）
├── Worker Celery（2 副本）
├── PostgreSQL（StatefulSet，1 副本 + 定时备份）
├── Redis（1 副本）
├── Ingress 配置域名 + TLS
├── Prometheus 监控 + Grafana 面板
├── EFK / Loki 日志收集
├── 完整的 Helm Chart
└── CI/CD：Git Push → 自动部署
```

---

## 四、核心命令速查

| 操作 | 命令 |
|------|------|
| 查看资源 | `kubectl get <kind>` / `kubectl describe <kind> <name>` |
| 创建/更新 | `kubectl apply -f <file>` |
| 删除 | `kubectl delete <kind> <name>` / `kubectl delete -f <file>` |
| 日志 | `kubectl logs <pod> [-c <container>] [-f]` |
| 进入容器 | `kubectl exec -it <pod> -- bash` |
| 端口转发 | `kubectl port-forward <pod> <local>:<remote>` |
| 扩缩容 | `kubectl scale deploy/<name> --replicas=N` |
| 滚动更新 | `kubectl set image deploy/<name> <container>=<image>` |
| 回滚 | `kubectl rollout undo deploy/<name>` |
| 资源使用 | `kubectl top pods` / `kubectl top nodes` |
| 查看事件 | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| YAML 导出 | `kubectl get <kind> <name> -o yaml` |

---

## 五、里程碑检查点

```
Week 1 结束：✓ 能创建 Pod/Deployment/Service，理解三者关系
Week 2 结束：✓ 能配置 Ingress 路由、持久化存储、网络策略
Week 3 结束：✓ 能配置 HPA 伸缩、Helm 打包、RBAC 权限
Week 4 结束：✓ 能独立完成一个应用的 K8s 生产级部署（含监控、日志、CI/CD）
```

---

## 六、推荐资源

| 类型 | 资源 |
|------|------|
| 官方文档 | kubernetes.io/docs（权威，建议看英文版） |
| 交互教程 | Kubernetes Basics（官方在线教程） |
| 在线练习 | Killercoda（免费 K8s 实验环境） |
| 书籍 | 《Kubernetes in Action》（经典入门） |
| 认证 | CKA / CKAD（证书含金量高，学完可以考） |
| 工具 | k9s（终端 UI）、Lens（桌面 GUI）、kubectx/kubens |
