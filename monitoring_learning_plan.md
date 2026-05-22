# 监控与可观测性学习路线

---

## 一、前置条件

- 已掌握 Docker 和 Kubernetes 基础（能跑容器、操作 Pod）
- 理解 Linux 基础（文件系统、进程、网络）
- 有 Python 基础（能写简单脚本）
- 了解 HTTP 协议基本概念

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ Prometheus    │    │ Grafana       │    │ 日志与链路    │    │ 综合实战      │
│ 指标采集      │    │ 可视化与告警  │    │ 追踪          │    │               │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：Prometheus — 指标采集与存储

**目标：理解 metrics 概念，能部署 Prometheus 并采集指标**

#### Day 1：可观测性三支柱

| 主题 | 内容 |
|------|------|
| 三大支柱 | **Metrics**（指标，数值型时间序列）/ **Logs**（日志，不可变事件记录）/ **Traces**（链路追踪，请求在分布式系统中的完整路径） |
| 为什么三者缺一不可 | 指标告诉你"有没有问题"，日志告诉你"具体是什么问题"，链路告诉你"问题出在哪个环节" |
| Google 四大黄金信号 | Latency（延迟）/ Traffic（流量）/ Errors（错误）/ Saturation（饱和度） |
| SLI / SLO / SLA | SLI = 服务水平指标（实际值），SLO = 服务水平目标（目标值），SLA = 服务水平协议（有合同约束） |

**练习：** 为你最熟悉的 Web 应用定义 4 个 SLI（延迟 p99、QPS、错误率、CPU 使用率），并为每个设定 SLO

#### Day 2：Prometheus 架构与数据模型

| 主题 | 内容 |
|------|------|
| Pull 模型 | Prometheus 主动拉取（scrape）target 的 `/metrics` 端点 |
| 时序数据 | `metric_name{label1="v1",label2="v2"} value timestamp` |
| 四种指标类型 | Counter（只增不减）/ Gauge（可增可减）/ Histogram（分桶统计）/ Summary（分位数） |
| 存储 | TSDB — 本地时序数据库，数据有保留期限 |

**练习：** 用 Docker 启动 Prometheus，访问 `localhost:9090`，熟悉 UI

```bash
docker run -d --name prometheus -p 9090:9090 prom/prometheus
```

#### Day 3：PromQL 入门

| 操作 | 示例 |
|------|------|
| 即时查询 | `http_requests_total` |
| 范围查询 | `http_requests_total[5m]` |
| 速率计算 | `rate(http_requests_total[5m])` — 每秒请求数 |
| 聚合 | `sum(rate(http_requests_total[5m])) by (status_code)` |
| Counter 专用 | `increase(http_requests_total[5m])` — 5 分钟内增量 |
| Gauge 专用 | `avg_over_time(cpu_usage[5m])` |

**练习：**
```promql
# 最近 5 分钟每秒请求速率
rate(http_requests_total[5m])

# 按状态码分组的错误率
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# 过去 1 小时的内存使用峰值
max_over_time(memory_usage_bytes[1h])

# 使用 histogram 计算 p99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

#### Day 4：Node Exporter — 主机指标

| 主题 | 内容 |
|------|------|
| node_exporter | 暴露 Linux 主机指标（CPU / 内存 / 磁盘 / 网络） |
| 部署方式 | 二进制部署或 DaemonSet |
| 关键指标 | `node_cpu_seconds_total`、`node_memory_MemAvailable_bytes`、`node_disk_io_time_seconds_total` |
| `--collector` | 启用额外采集器（systemd / processes 等） |

**练习：** 用 Docker 启动 node_exporter，配置 Prometheus 去 scrape 它

```bash
docker run -d --name node-exporter --net=host \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/rootfs:ro \
  prom/node-exporter \
  --path.rootfs=/rootfs \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys
```

#### Day 5：应用指标 — 给你的 Python 应用加监控

| 主题 | 内容 |
|------|------|
| prometheus_client | Python 官方 Prometheus 客户端库 |
| 默认指标 | `process_*`、`python_gc_*` |
| 自定义 Counter | 记录 API 调用次数 |
| 自定义 Histogram | 记录请求处理时间 |
| 暴露 `/metrics` | 通过 HTTP 端点暴露给 Prometheus scrape |

**练习：**
```python
from prometheus_client import Counter, Histogram, generate_latest, start_http_server

REQUEST_COUNT = Counter('app_requests_total', 'Total requests',
                        ['method', 'endpoint', 'status'])
REQUEST_DURATION = Histogram('app_request_duration_seconds',
                             'Request duration', ['method', 'endpoint'])

# 在 Flask/FastAPI 的 middleware 中：
# REQUEST_COUNT.labels(method='GET', endpoint='/api', status='200').inc()
# REQUEST_DURATION.labels(method='GET', endpoint='/api').observe(elapsed)

start_http_server(8000)                    # 在 8000 端口暴露 /metrics
```

#### Day 6：服务发现与 relabeling

| 主题 | 内容 |
|------|------|
| 静态配置 | `static_configs` — 写死 target 地址 |
| 文件服务发现 | `file_sd_configs` — JSON/YAML 文件 |
| Kubernetes 服务发现 | `kubernetes_sd_configs` — 自动发现 Pod/Service/Endpoints |
| relabel_configs | scrape 前对 label 做增删改 |
| metric_relabel_configs | 存储前对 metric label 做增删改（节省存储） |

**练习：** 配置 Prometheus 用 K8s 服务发现自动 scrape 集群中带 annotation `prometheus.io/scrape: "true"` 的 Pod

#### Day 7：第一周综合练习

```
目标：搭建一个完整的指标采集链路

任务：
1. Docker Compose 编排以下服务：
   ├── Prometheus (scrape 中心)
   ├── Node Exporter (主机指标)
   ├── 自写的 Python Web 应用 (带 /metrics，暴露业务指标)
   └── 自写的 Python Worker 应用 (带 /metrics)

2. 配置 Prometheus：
   - scrape_interval: 15s
   - 采集所有 target
   - 为 job 标签设置有意义的值

3. 用 PromQL 完成以下查询（用 curl 请求 Prometheus API）：
   - 各实例 CPU 使用率
   - Web API 的 QPS（按 endpoint 分组）
   - Web API 的 p95 延迟
   - 5xx 错误比例
```

---

### 第2周：Grafana — 可视化与告警

**目标：能用 Grafana 创建 Dashboard、配置告警规则**

#### Day 8：Grafana 入门

| 主题 | 内容 |
|------|------|
| Grafana 角色 | 数据可视化 + 告警引擎（不存数据，只展示和告警） |
| Data Source | 对接 Prometheus / InfluxDB / Elasticsearch 等 |
| Dashboard | 多个 Panel 组成一个看板 |
| Panel | 单个图表（时间序列 / 统计值 / 表格 / 热力图） |

**练习：** Docker 启动 Grafana，添加 Prometheus 为 Data Source

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

#### Day 9：创建第一个 Dashboard

| 主题 | 说明 |
|------|------|
| Time Series Panel | 折线图 — 最常用 |
| Stat Panel | 单个数字 — 当前值/平均值 |
| Gauge Panel | 仪表盘 — 显示用量百分比 |
| Table Panel | 表格 — 多维度明细 |
| 变量（Variables） | `$instance`、`$job`，让 Dashboard 可交互 |
| 模板化 | 带变量的查询可跨环境复用 |

**练习：** 创建一个"主机概览"Dashboard，包含 CPU / 内存 / 磁盘 / 网络四个 Panel，并添加 `$instance` 变量

#### Day 10：Grafana 高级图表

| Panel 类型 | 使用场景 |
|------------|----------|
| **Heatmap** | Histogram 数据可视化，看延迟分布 |
| **Stat (sparkline)** | 当前值 + 小趋势图 |
| **Bar Gauge** | 横向或纵向条形图 |
| **State Timeline** | 状态时间线（如 Pod 状态变迁） |
| **Geomap** | 地理分布图 |

**练习：** 用 Heatmap Panel 可视化 HTTP 请求延迟分布（histogram_quantile）

#### Day 11：Alertmanager — 告警

| 主题 | 内容 |
|------|------|
| Alertmanager 架构 | Prometheus 评估规则 → 推送告警到 Alertmanager → 分组/抑制/静默 → 通知 |
| 告警规则 | PromQL + `for:`（持续多久才触发）+ annotations |
| 路由（Routing） | 按 label 分组，不同路由不同接收者 |
| 分组（Grouping） | 同类告警合并成一条通知 |
| 抑制（Inhibition） | 如果 A 告警在 firing，B 就静音 |
| 静默（Silence） | 已知维护窗口内不通知 |

**练习：**
```yaml
# alerting_rules.yml
groups:
  - name: app_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High 5xx rate on {{ $labels.instance }}"
          description: "5xx rate is {{ $value }} for the last 5 minutes"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency > 1s on {{ $labels.instance }}"
```

#### Day 12：告警通知渠道

| 接收者 | 配置要点 |
|--------|----------|
| **Webhook** | 最通用，可对接任意系统 |
| **飞书 / Slack** | Webhook URL + 消息模板 |
| **Email** | SMTP 配置 |
| **PagerDuty** | 专业的 On-Call 管理 |

**练习：** 配置 Alertmanager 发送告警到飞书机器人 webhook

#### Day 13：Recording Rules & Federation

| 主题 | 内容 |
|------|------|
| Recording Rules | 预计算常用查询，加速 Dashboard 渲染 |
| 语法 | 同 PromQL，定时执行并存入新 metric |
| Federation | Prometheus 层级架构：下级 Prometheus 聚合给上级 |
| 适用场景 | 多集群 / 多数据中心，分层聚合 |

**练习：** 创建一条 recording rule 预计算 `job:http_errors:rate5m`，然后在 Grafana 中使用这个预计算指标

#### Day 14：第二周综合练习

```
目标：搭建完整的可视化与告警体系

任务：
1. 给第1周的 Python 应用创建 3 个 Grafana Dashboard：
   - 概览 Dashboard：QPS / 延迟 p50,p95,p99 / 错误率 / 在线实例数
   - 业务 Dashboard：按 endpoint 的业务指标
   - 资源 Dashboard：CPU / Memory / GC

2. 配置 4 条告警规则：
   - 5xx 错误率超过 5% 持续 5min → critical
   - p99 延迟超过 1s 持续 10min → warning
   - 实例数小于 2 → critical
   - 磁盘使用率超过 85% → warning

3. 配置 Alertmanager：
   - 按 severity 路由（critical → 飞书 + 邮件, warning → 飞书）
   - 同类告警 5 分钟内合并
   - 配置维护窗口静默规则

4. 将 Dashboard 导出为 JSON，提交到 Git
```

---

### 第3周：日志与链路追踪

**目标：理解日志系统的设计，能搭建日志聚合和分布式追踪**

#### Day 15：日志 — 从 printf 到结构化

| 主题 | 内容 |
|------|------|
| 传统日志的问题 | 格式不统一、分散在各机器、grep 效率低 |
| 结构化日志 | JSON 格式，可被机器解析和索引 |
| 日志级别 | DEBUG / INFO / WARN / ERROR（含 context 字段） |
| 12-Factor App 原则 | 日志只写 stdout，由平台负责收集 |
| Python logging | `structlog` — 结构化日志库 |

**练习：** 将 Python 应用的日志从 `print` 改为 `structlog` JSON 输出

```python
import structlog
import json

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer()
    ]
)
log = structlog.get_logger()
log.info("request_processed", method="GET", path="/api", status=200, duration_ms=45)
```

#### Day 16：Loki — 日志聚合

| 主题 | 内容 |
|------|------|
| Loki 的设计哲学 | 只索引 label，不索引日志内容（低成本，对 K8s 友好） |
| 架构 | Promtail（采集）→ Loki（存储）→ Grafana（查询） |
| LogQL | 类 PromQL 的日志查询语言 |
| Label | `{app="web", env="prod"}` |

**练习：** 用 Docker Compose 启动 Loki + Promtail，采集 Python 应用的 stdout 日志

```yaml
# compose 骨架
services:
  loki:
    image: grafana/loki
  promtail:
    image: grafana/promtail
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yml:/etc/promtail/config.yml
  grafana:
    image: grafana/grafana
```

#### Day 17：LogQL 查询

| 查询类型 | 示例 |
|----------|------|
| 过滤 | `{app="web"} |= "ERROR"` |
| 排除 | `{app="web"} != "DEBUG"` |
| 正则 | `{app="web"} |~ "(?i)error|fail"` |
| 聚合 | `rate({app="web"} |= "ERROR" [5m])` |
| 提取 | `{app="web"} | json | line_format "{{.message}}"` |

**练习：** 在 Grafana 的 Explore 视图中，用 LogQL 查询：最近 1 小时内 web 服务的所有 ERROR 日志，并按 endpoint 分组统计数量

#### Day 18：分布式追踪概念

| 主题 | 内容 |
|------|------|
| 为什么需要 Tracing | 一个请求经过 N 个微服务，定位瓶颈需要"请求的全景视图" |
| Span | 一个操作单元（有名称、开始时间、持续时间、tag） |
| Trace | 一组 Span 的树形结构，共享一个 trace_id |
| Context Propagation | trace_id 和 span_id 通过 HTTP header（如 W3C TraceContext）传递 |
| Sampling | 全量采集成本高，需采样策略 |

**练习：** 画一个请求经过 "Nginx → Web API → DB + Redis" 的 Span 树

#### Day 19：OpenTelemetry + Jaeger

| 主题 | 内容 |
|------|------|
| OpenTelemetry | CNCF 可观测性标准，统一 API/SDK |
| OTLP | OpenTelemetry Protocol，传输数据到后端 |
| Jaeger | 分布式追踪后端，存储和可视化 Trace |
| Auto-instrumentation | Python Agent 自动注入 tracing |

**练习：** 为 Python 应用添加 OpenTelemetry SDK，将 trace 发送到 Jaeger

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor

# 初始化
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

# 自动插桩
FlaskInstrumentor().instrument_app(app)

# 手动创建 Span
with tracer.start_as_current_span("db_query") as span:
    span.set_attribute("db.statement", sql)
    result = db.execute(sql)
```

#### Day 20：三个支柱的关联 — 用 Exemplar 打通

| 主题 | 内容 |
|------|------|
| Exemplar | 在 metric 中嵌入 trace_id，从指标直接跳到 trace |
| Loki + trace_id | 日志中带 trace_id，从日志跳到 trace |
| 统一的可观测性 | Metrics → 发现异常 → 看 Exemplar → 找到 Trace → 看关联日志 |

**练习：** 在 Grafana Dashboard 中配置 Exemplar，实现从延迟指标图表一键跳转到 Jaeger Trace 详情

#### Day 21：第三周综合练习

```
目标：搭建完整的三支柱可观测性平台

任务：
1. Docker Compose 编排：
   ├── Prometheus (指标)
   ├── Grafana (可视化)
   ├── Alertmanager (告警)
   ├── Loki + Promtail (日志聚合)
   ├── Jaeger + OpenTelemetry Collector (链路追踪)
   ├── Python Web App (带 metrics / structured logging / tracing)
   └── Python Worker App (带 metrics / structured logging / tracing)

2. 验证三支柱的关联：
   - 在 Grafana 中从指标面板通过 Exemplar 跳转到 Jaeger
   - 在 Jaeger Trace 中看到关联的日志（trace_id 可关联到 Loki）
   - 在 Loki 日志中通过 trace_id 反查完整调用链

3. 注入一个慢查询故障，在 Grafana 中观察到：
   - p99 延迟告警触发
   - 从告警跳转到异常 Trace
   - 从 Trace 定位到具体的 DB 查询 Span
   - 从 Span 关联日志看到具体的 SQL 语句
```

---

### 第4周：综合实战 — 生产级可观测性平台

**目标：构建一个可用于生产环境的监控体系，产出可复用的配置和文档**

#### Day 22：K8s 环境部署 Prometheus Stack

| 主题 | 内容 |
|------|------|
| kube-prometheus-stack | Helm Chart，一键部署 Prometheus + Grafana + Alertmanager |
| Prometheus Operator | CRD（ServiceMonitor / PodMonitor）管理 scrape 配置 |
| ServiceMonitor | K8s 原生方式声明要采集哪些 Service |
| 常用 exporters | kube-state-metrics（集群状态）、cAdvisor（容器资源）、node-exporter |

**练习：** 用 Helm 在 K8s 集群中安装 kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --set grafana.adminPassword=admin
```

#### Day 23：K8s 应用监控

| 主题 | 内容 |
|------|------|
| PodMonitor vs ServiceMonitor | 直接监控 Pod 或通过 Service 间接监控 |
| annotation 自动发现 | 给 Pod 打 annotation，Prometheus 自动发现 |
| 常用 K8s 指标 | `kube_pod_status_phase`、`kube_deployment_status_replicas`、`container_memory_working_set_bytes` |

**练习：** 部署一个 Python 应用到 K8s，创建 ServiceMonitor 让 Prometheus 自动采集

#### Day 24：SLO Dashboard 设计

| 主题 | 内容 |
|------|------|
| SLI 选型 | Latency / Availability / Throughput / Error Budget |
| Error Budget | 1 - SLO，即可接受的错误预算 |
| Burn Rate | 错误预算消耗速率，用于告警 |
| SLO Dashboard 模板 | 显示 SLI 当前值 / SLO 目标 / 错误预算剩余 / 消耗速率 |

**练习：** 为你的应用创建 SLO Dashboard，包含 4 个 SLI 的面板 + 错误预算燃尽图

#### Day 25：K8s 日志采集（Loki + Promtail / Grafana Agent）

| 主题 | 内容 |
|------|------|
| Promtail on K8s | DaemonSet 部署，采集所有节点容器日志 |
| 日志 label 提取 | 从 K8s 元数据自动添加 namespace / pod / container label |
| Grafana Agent | 统一采集 agent（替代 Promtail + node_exporter，减少组件数） |
| 日志保留策略 | retention_period 配置 |

**练习：** 在 K8s 中部署 Loki + Grafana Agent，实现全集群日志聚合

#### Day 26：告警分级与 On-Call 流程

| 主题 | 内容 |
|------|------|
| 告警分级 | Critical（立即处理）/ Warning（工作时间处理）/ Info（不通知，仅记录） |
| 告警抑制 | 主机宕了 → 抑制该主机上所有服务告警 |
| On-Call 轮值 | 排班制度，确保每时每刻有人在 |
| Runbook | 每条告警应该有对应的处理文档 |
| 告警复盘 | 分析误报率，持续优化告警规则 |

**练习：** 编写 3 条告警的 Runbook（现象、排查步骤、修复方案、升级路径）

#### Day 27：Dashboard as Code — Grafana Provisioning

| 主题 | 内容 |
|------|------|
| Dashboard JSON Model | Grafana Dashboard 的完整 JSON 结构 |
| Provisioning | 通过文件自动创建 Data Source + Dashboard |
| grafonnet / grafanalib | 用代码生成 Dashboard JSON（像 IaC 一样管理 Dashboard） |
| Git 版本管理 | Dashboard 的变更走 Git PR 流程 |

**练习：** 用 Grafana Provisioning 配置自动加载 Data Source 和 Dashboard，实现 Grafana 启动即可用

#### Day 28：第四周综合练习

搭建一个完整的生产级可观测性平台：

```
集群级监控（K8s）：
├── kube-prometheus-stack（Prometheus + Grafana + Alertmanager）
├── Loki + Grafana Agent（日志聚合）
├── Jaeger（链路追踪）
└── 自写 Python 应用（多副本，带完整 instrumentation）

产出物：
1. 一个 SLO Dashboard（SLI 趋势 + 错误预算燃尽）
2. 一个 App Dashboard（QPS / 延迟 / 错误 / 资源）
3. 一个 日志 Dashboard（日志量趋势 + 错误日志 Top N）
4. 告警规则文件（含 5 条以上规则，按 severity 分级）
5. Alertmanager 配置（分组 + 抑制 + 路由）
6. 3 条告警 Runbook
7. 所有 Dashboard JSON 提交到 Git
8. 一份架构说明文档（组件拓扑 + 数据流向）
```

---

## 四、里程碑检查点

```
Week 1 结束：✓ 能解释 metrics/logs/traces 的区别和关系
             ✓ 能部署 Prometheus，用 PromQL 写出 10 种以上查询
             ✓ 能为 Python 应用添加自定义 metrics
Week 2 结束：✓ 能创建包含 5 种 Panel 类型的 Grafana Dashboard
             ✓ 能配置 Alertmanager 告警（分组、路由、静默）
Week 3 结束：✓ 能搭建 Loki 日志聚合，用 LogQL 查询
             ✓ 能整合 OpenTelemetry + Jaeger 实现分布式追踪
Week 4 结束：✓ 能在 K8s 部署完整可观测性栈
             ✓ 能产出 SLO Dashboard 和告警 Runbook
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 必读书籍 | Google SRE Book — sre.google/books（中文版免费） |
| 必读书籍 | 《Prometheus 监控实战》（Prometheus: Up & Running 中文版） |
| 文档 | Prometheus 官方文档 — prometheus.io/docs |
| 文档 | Grafana 官方文档 — grafana.com/docs |
| 文档 | OpenTelemetry 官方文档 — opentelemetry.io/docs |
| 实践指南 | Grafana Play — play.grafana.org（在线体验各种 Dashboard） |
| 社区 | Grafana Dashboard 市场 — grafana.com/grafana/dashboards |
| 视频 | "Monitoring Modern Applications" — Datadog 出品的概念讲解 |
| 工具 | `promtool`（Prometheus 配置校验工具） |
