# QuantDinger 项目归档分析（2026-09）

> 归档目的：为后续开发提供一次完整、可检索的项目结构与运行机制快照。本文档由 AI 开发代理在 2026-09-13 通过对仓库的只读全面分析生成，后续开发如修改架构请同步更新本文档。
>
> 版本基线：`VERSION = 5.0.17`，MCP 包 `quantdinger-mcp = 0.6.2`（版本独立）。HEAD 提交：`24f1d6a`。

## 1. 仓库全景

```
QuantDinger/
├── .github/workflows/            5 条 CI 流水线（basic / docker-publish / mcp / openapi / security）
├── backend_api_python/           Flask 后端与全部后端进程（app + migrations + scripts + tests）
├── docs/                         文档（architecture / deployment / trading / api / agent / analysis）
├── mcp_server/                   MCP 服务器（独立 PyPI 包 quantdinger-mcp）
├── ops/                          Prometheus / Grafana / Alertmanager 配置
├── scripts/                      仓库级脚本（check_version / check_docs / check_mojibake / bump_version）
├── docker-compose*.yml           5 个 compose 文件（core / ghcr / build / production / observability）
├── install.sh / install.ps1      GHCR 零仓库一键安装器
└── VERSION                       canonical 版本文件（与 backend_api_python/VERSION 校验一致）
```

## 2. 技术栈与关键依赖

| 领域 | 选型 |
|---|---|
| Web 框架 | Flask 3.1 + flask-smorest（自动 OpenAPI）+ marshmallow |
| WSGI | Gunicorn（gthread，1 worker × 4 线程，`gunicorn_config.py`） |
| 任务队列 | Celery 5.6（`task_acks_late`、`reject_on_worker_lost`、Asia/Shanghai） |
| 数据库 | PostgreSQL 18，原生 psycopg2（无 ORM，无 Alembic） |
| 缓存/队列 | 双 Redis：cache（可逐出）+ redis-jobs（持久 Celery broker/result，AOF+noeviction） |
| 行情 | ccxt、yfinance、akshare、finnhub、exchange-calendars、TA-Lib |
| LLM | litellm（OpenRouter/OpenAI/Gemini/DeepSeek/Grok/AtlasCloud/Custom/MiniMax） |
| 报告 | pypdf、reportlab |
| 监控 | prometheus-client |
| 安全 | PyJWT、cryptography、bcrypt、pyotp、Gitleaks/Bandit/pip-audit/CodeQL（CI） |
| 券商 | ib_insync（IBKR）、alpaca-py |

## 3. 进程模型（QD_PROCESS_ROLE 决定责任边界）

同一后端镜像按环境变量 `QD_PROCESS_ROLE` 运行 6 种进程。**职责边界是架构的灵魂**：

| 角色 | 责任 | 入口 |
|---|---|---|
| `migration` | 应用 schema 后退出 | `app/commands/migrate.py`（fail-fast） |
| `api`/`backend` | HTTP、鉴权、校验、提交持久化命令 | `backend_api_python/run.py` → gunicorn |
| `trading` | 策略运行时、挂单、券商会话、对账、租约 | `app/commands/trading_worker.py` |
| `scheduler` | 组合/部署/支付/信号调度（leader 选举） | `app/commands/scheduler.py` |
| `celery` + beat | 有限可重试异步任务 | `app/celery_app.py` |

规则：
- HTTP 进程**不得**拥有交易循环、券商专属行为或大型 DB 工作流。
- 长驻行为放在 `app/workers/`（`TradingWorker`、`LeaseHeartbeat`）；有限任务放 `app/tasks/`；进程入口在 `app/commands/`。
- 策略启停写入 `qd_strategy_commands` 表 → `TradingWorker` 认领执行；`qd_strategy_runtime_leases` / `qd_process_leases` + 独立心跳连接保证崩溃恢复与 fencing。

## 4. 分层与目录归属

### 4.1 app/ 顶层目录

| 目录 | 职责 | 关键文件 |
|---|---|---|
| `routes/` | HTTP 门面（人类 + agent 双面） | `agent_v1/`（16 个路由）、strategy.py、ai_chat.py |
| `services/` | 领域工作流（~240 文件，最大） | strategy_runtime/、strategy_v2/、live_trading/、grid/、ai_* |
| `data_sources/` | 行情/K线数据源 | base.py、factory.py、crypto.py、us_stock.py、errors.py |
| `data_providers/` | 研究/展示聚合数据 | us_research.py、hk_research.py、sentiment、news、macro |
| `tasks/` | Celery 有限任务 | agent_jobs.py、fast_analysis.py、maintenance.py |
| `workers/` | 进程内长驻循环 | trading.py、lease_heartbeat.py |
| `commands/` | 进程入口 | migrate.py、trading_worker.py、scheduler.py、worker_health.py |
| `config/` | env 配置类（metaclass 动态读环境变量） | settings.py、api_keys.py、database.py、data_sources.py |
| `openapi/` | flask-smorest / OpenAPI | register.py（25 前缀→tag）、schemas/ |
| `observability/` | request_id + Prometheus 指标埋点 | metrics.py、http.py、context.py |
| `markets/` | 市场模块注册中心 | registry.py（7 个市场） |
| `runtime/` | 进程角色模型 | roles.py、process.py |
| `utils/` | 基础设施 | db.py、config_loader.py、auth.py、credential_crypto.py、risk_guard.py |
| `professional_report/` | AI 专业报告管线 | builder.py、contracts.py、features/、providers/ |

### 4.2 data_sources vs data_providers

- `data_sources` = **计算所需**的历史/实时行情（K 线统一形状 time/open/high/low/close/volume），供回测、实盘、K线图表、AI 统计使用。
- `data_providers` = **对外展示/研究**的聚合快照（宏观、新闻、情绪、热力），带 `cached_or_compute`（single-flight + SWR）与 TTL 缓存，防上游接口被打爆。

### 4.3 HTTP API 面

- **人类面** `/api/*`：flask-smorest 自动 OpenAPI，`app/openapi/register.py` 定义 25 个前缀→tag 映射；认证走 `/api/auth`（登录/MFA/OAuth/注册/找回密码）。
- **Agent 面** `/api/agent/v1`：独立 token 鉴权（`agent_v1/_security.py`）+ `qd_agent_audit` 审计；能力类 R/W/B/N/C/T 逐路由强制；所有变更工具强制幂等键；任务类端点异步执行并可用 SSE 流式返回进度。
- MCP 服务器（`mcp_server/`）是 Agent Gateway 的**薄封装**（58 个工具），REST 是唯一真实源；工具带 readOnly/destructive/idempotent 注解，错误响应统一脱敏后以 `isError=True` 返回。

### 4.4 交易执行链路

```
StrategySignal → SignalGate → PositionSizer → OrderIntentBuilder
      → OrderIntentService（idempotency_key 幂等）→ qd_strategy_order_intents
      → TradingExecutor / StrategyV2LiveSession
      → StrategyV2OrderGateway（pending→processing→sent→syncing 状态机）
      → live_trading.execution.place_order_from_signal
      → LiveOrderPhaseAdapter → create_client(exchange) → 下单
```

交易所组件：Binance / OKX / Bitget / Bybit / Gate / HTX（crypto）；IBKR / Alpaca（券商）。网格策略经 `GridEngine` + `GridRestingRunner` 常驻挂单、停止时清理挂单。

### 4.5 AI 链路

```
ai_chat（Copilot，/api/ai，SSE 流式）
  → llm.py（多供应商 LLMService，可重试错误码）
  → ai_market_query（确定性规划器：仅 7 类任务 + 受控指标；LLM 只做语义提示，数值全部由代码计算，防幻觉）
  → professional_report（证据优先：snapshot/features/narrative/risk/quality）
  → FastAnalysisService v3（MarketDataCollector → LLM 单次调用 → 报告信封）
  → analysis_memory + reflection（事后核对收益、写回 was_correct/actual_return_pct、触发校准）
```

策略 AI：`strategy_ai_generation`（生成）→ `compile_strategy_v2`（校验）→ `strategy_ai_behavior`（小数据冒烟回测验证可执行性）。

### 4.6 数据库与迁移

- 无 ORM：psycopg2 连接池（`app/utils/db_postgres.py`），自动容量探测（按 PG `max_connections` × worker 均摊）、指数退避等待、`?`→`%s` 占位符转换、无 RETURNING 的 INSERT 自动 `RETURNING id`。
- 迁移：`migrations/` 目录 13+ 个增量 `2026MMDD_*.sql` + `init.sql`（schema）+ 2 个种子文件。应用启动自动 `init_database()`，对三个组件算 SHA-256 存 `qd_bootstrap_migrations` 账本，仅内容变化时执行（幂等）。
- 表命名 `qd_*`（用户/积分/策略/订单/审计/命令/租约/心跳/意图等）。

## 5. 部署模型

| 文件 | 用途 |
|---|---|
| `docker-compose.yml` | 核心本地栈（12 services）；后端从源码构建，前端/mobile 从 GHCR；`mcp` 服务 profile-gated |
| `docker-compose.ghcr.yml` | 零仓库 GHCR 栈（install.sh/ps1 使用） |
| `docker-compose.build.yml` | 前端/mobile 本地构建覆盖层 |
| `docker-compose.production.yml` | 生产加固：非 root、read_only、cap_drop ALL、.env 只读挂载、资源限制 |
| `docker-compose.observability.yml` | Prometheus + Grafana + Alertmanager + 3 exporters |

- 端口：Web 8888、Mobile H5 8889、Backend 5000、Grafana 3000、Prometheus 9090、Alertmanager 9093（默认全回环绑定）。
- `ops/prometheus/alerts.yml` 7 条告警；Grafana 预置 "QuantDinger Runtime Overview" 仪表盘。
- 安装器 `install.sh`/`install.ps1`：下载 compose + 交互收集凭据（禁 123456）+ 生成 secrets + 拉起栈 + 健康门禁。

## 6. 测试与 CI

### 测试组织（backend_api_python/tests/）

- 253 个根级 `test_*.py`，平铺按业务模块命名；`pytest.ini`（timeout=120；markers: integration、stress）。
- `fixtures/`：契约 JSON；`integration/`：需真实 PG 的检查脚本；`release_gate/`：实盘执行语义门禁。
- 常规运行：`python -m pytest -m "not integration and not stress" --ignore=tests/release_gate -q`

### CI（.github/workflows/）

| Workflow | 内容 |
|---|---|
| basic-ci | quality（ruff+compile+check_docs+backend_quality_check）+ tests（PG+Redis 服务矩阵）+ compose-check + version-check |
| docker-publish | v* tag → GHCR 多架构镜像 + GitHub Release |
| mcp-ci | mcp_server 矩阵测试（3.10/3.12/3.13）+ build |
| openapi-ci | openapi.yaml 导出一致性 + Spectral lint + oasdiff 兼容性 |
| security-ci | pip-audit + Bandit + Gitleaks + CodeQL |

## 7. 关键运维/安全注意点

- `SECRET_KEY` / `CREDENTIAL_ENCRYPTION_KEY` / `ADMIN_PASSWORD` 缺失或不安全时启动 fail-fast。
- 凭据经 `CREDENTIAL_ENCRYPTION_KEY` 加密存储；Agent token 哈希化、作用域限制、限流、审计。
- Agent 实盘交易默认 paper-only；需 token scope + `paper_only=false` + `AGENT_LIVE_TRADING_ENABLED=true` 三条件齐备。
- `/app/.env` 含密码与密钥，必须保持 UID 10001 可写、mode 600；生产 overlay 下为只读挂载。
- 高频触碰面（实盘、网格、撤单、凭据、计费、迁移）改动必须配测试并跑 release_gate。

## 8. 修改影响面速查

| 变更 | 主位置 | 联动 |
|---|---|---|
| HTTP 端点 | app/routes/ | app/openapi/ + 契约测试 + API 文档 |
| 业务工作流 | app/services/ | service 测试 |
| 交易所/券商 | app/services/live_trading/ | 凭据策略 + 适配器测试 + 文档 |
| 行情数据源 | app/data_sources/ | provider 聚合 + 缓存键 + 测试 |
| 异步任务 | app/tasks/ | celery_app.py + 队列路由 + 测试 |
| 长驻进程 | app/workers/ + commands/ | Compose command + 健康检查 + ownership 测试 |
| schema | migrations/ | 迁移测试 + 文档 |
| 指标/告警 | app/observability/ + ops/ | 仪表盘 + 告警规则 + 文档 |
| MCP 工具 | mcp_server/ | Agent scope + 安全测试 + agent 文档 |
| API 契约 | app/openapi/ | docs/api/openapi.yaml + docs/agent/agent-openapi.json |