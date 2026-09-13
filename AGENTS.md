# AGENTS.md — QuantDinger 项目开发指令

本仓库是 **QuantDinger**（自托管 AI 量化交易系统，Open Byte Inc. 出品）的分支。本文件为 AI 开发代理（opencode）在本仓库内工作时必须遵守的指令与约束。

---

## 1. 项目概述

- **定位**：开源 AI Trading OS。把交易想法变成 Python 策略 → 回测 → 纸上交易 → 实盘执行 → 监控，全部自托管。
- **技术栈**：Python 3.12 / Flask 3.1 + flask-smorest / Gunicorn / Celery 5.6 / PostgreSQL 18 / Redis 8 / Prometheus + Grafana。
- **核心原则**：本地优先、自我托管；市场数据、策略代码、券商凭据、部署均归运营者掌控。**不提供投资建议。**
- **版本**：仓库根 `VERSION` 为 canonical；镜像 tag `v*` 触发 GHCR 发布；MCP 包 `quantdinger-mcp` 版本独立于主版本。
- **上游**：OpenByteInc/QuantDinger。本仓库 origin 为 `https://github.com/LCecho/QuantDinger.git`。

## 2. 架构速览（必须理解后再动手）

### 2.1 进程角色（`QD_PROCESS_ROLE` 决定责任边界）

| 进程 | 职责 | 入口 |
| --- | --- | --- |
| `migration` | 应用数据库 schema，退出 | `app/commands/migrate.py` |
| `backend` (api) | HTTP、鉴权、校验、**只提交持久化命令** | `backend_api_python/run.py` + gunicorn |
| `trading-worker` | 策略运行时、挂单、券商会话、对账 | `app/commands/trading_worker.py` |
| `scheduler-worker` | 组合、部署、支付、信号调度 | `app/commands/scheduler.py` |
| `celery-worker`/`celery-beat` | 有限的、可重试的异步任务 | `app/celery_app.py` |

**硬性规则**：HTTP 进程 `app/routes` 只做「解析/校验/委托」，**不得**拥有交易循环、券商专属行为或大型 DB 工作流。长驻行为必须放在对应 worker 进程。

- 有限可重试工作 → `app/tasks/`（Celery）
- 进程内长驻循环/租约 → `app/workers/`
- 进程启动脚本 → `app/commands/`
- 策略启停不会话式依赖内存 → 写 `qd_strategy_commands` 表由 `TradingWorker` 认领执行（租约 + 心跳 + fencing token）

### 2.2 分层与归属

```
routes/        HTTP 门面（人类 /api 与 agent /api/agent/v1 双面）
  └─ services/         领域工作流与第三方集成（最大目录，~240 文件）
       ├─ strategy_runtime/   策略运行时管线（信号→仓位→订单意图）
       ├─ strategy_v2/        Strategy API V2（contract/runtime/service/live_execution）
       ├─ live_trading/       交易所归一化适配器（binance/okx/bitget/bybit/gate/htx）
       ├─ grid/               网格交易引擎
       └─ ai_* / professional_report/ / billing / community ...
  data_sources/    行情/K线数据源（BaseDataSource 统一行形状 time/open/high/low/close/volume）
  data_providers/  聚合展示数据（宏观/新闻/情绪/热力，SWR 缓存，防上游打爆）
  tasks/ workers/ commands/ runtime/  observability/ utils/
```

- **data_sources = 计算所需行情**；**data_providers = 展示/研究聚合快照**。不要混用。
- **`app/routes` 不得 import flask `request` 到 services**；规则约束即架构。
- **无 ORM**：原生 psycopg2 + 手写 SQL。迁移 = 幂等 SQL + 启动校验和账本（`qd_bootstrap_migrations`），没有 Alembic/Flyway。

### 2.3 交易执行链路（改动涉及面广，务必看全链路）

```
StrategySignal → SignalGate → PositionSizer → OrderIntentBuilder
      → OrderIntentService(idempotency_key 幂等) → qd_strategy_order_intents
      → TradingExecutor / StrategyV2LiveSession
      → StrategyV2OrderGateway（pending→processing→sent→syncing 状态机）
      → live_trading.execution.place_order_from_signal
      → LiveOrderPhaseAdapter → create_client(exchange) → 下单
```

### 2.4 市场数据与 AI 链路

- 行情：`app/data_sources/`（crypto/us_stock/cn_stock/hk_stock/forex/futures/moex）→ `DataSourceFactory` 按市场分派 → 缓存 + 熔断 + 限流。
- AI：`ai_chat`（Copilot）→ `llm.py`（多供应商）→ `ai_market_query`（确定性数据，LLM 只做语义提示，数值必须由代码计算，防幻觉）→ `professional_report`（证据优先报告）→ `analysis_memory` + `reflection`（闭环校准）。
- Agent：`/api/agent/v1` 独立网关（token 鉴权 + 审计），`mcp_server/` 是其薄封装。

## 3. 开发工作流

### 3.1 提交前必须执行的质量检查（后端）

```bash
cd backend_api_python
python -m compileall -q app scripts tests
ruff check app scripts tests              # 规则见 pyproject.toml（line-length 120）
python scripts/backend_quality_check.py
python -m pytest -m "not integration and not stress" --ignore=tests/release_gate -q
python -m pytest tests/release_gate/test_live_execution_release_gate.py -q   # 涉及实盘执行时必跑
```

仓库级检查：

```bash
python scripts/check_version.py
python scripts/check_docs.py
python scripts/check_mojibake.py
docker compose -f docker-compose.yml config -q
docker compose -f docker-compose.yml -f docker-compose.production.yml -f docker-compose.observability.yml config -q
```

### 3.2 改动归属表（改哪里、同步改哪里）

| 变更 | 主要位置 | 通常还需同步 |
| --- | --- | --- |
| 新增/修改 HTTP 端点 | `backend_api_python/app/routes/` | `app/openapi/`、路由/契约测试、API 文档 |
| 新增业务工作流 | `backend_api_python/app/services/` | 聚焦的 service 测试 |
| 新增交易所/券商集成 | `app/services/live_trading/` 或对应券商包 | 凭据策略、适配器测试、文档 |
| 新增行情数据源 | `app/data_sources/` | 供应商聚合、缓存键、测试 |
| 新增有限异步任务 | `app/tasks/` | `celery_app.py`、队列路由、任务测试 |
| 修改长驻进程行为 | `app/workers/`、`app/commands/`、`app/runtime/` | Compose command、健康检查、ownership 测试 |
| 修改 DB schema | `backend_api_python/migrations/` | migration/release-gate 测试、文档 |
| 新增指标/告警 | `app/observability/` 与 `ops/` | 仪表盘、告警规则、可观测性文档 |
| 新增 MCP 工具 | `mcp_server/src/quantdinger_mcp/` | Agent Gateway scope、安全测试、agent 文档 |
| 修改 API 契约 | `app/openapi/` | `docs/api/openapi.yaml`、`docs/agent/agent-openapi.json`、CI 一致性 | 

### 3.3 测试规范

- pytest 配置在 `backend_api_python/pytest.ini`（`timeout=120`；标记 `integration`、`stress`）。
- 测试命名平铺在 `tests/test_*.py`，按业务模块一一对应。
- 涉及 DB/外部依赖的用例要能用 fixture `app`/`client`（`tests/conftest.py`）隔离。
- 不要给违反约束的改动写「通过型」测试糊弄过去。

## 4. 硬性开发约束（不可违反）

### 4.1 删除操作需严格二次确认
- 任何**删除文件/目录/代码块**的操作，在命令执行前必须向用户重复确认，说明删除对象与影响面。
- 本仓库根（`D:\project\QuanterDinger`）以外的任何路径更需严格二次确认。
- 确认方式是明确列出将被删除的路径清单，等待用户显式同意后再执行。

### 4.2 每次验证完成的任务必须提交 GitHub（含删除）
- 每完成一个**通过验证**的任务（含 lint / 测试 / build 通过），立即提交到 GitHub。
- **删除操作也必须提交**（`git rm`/删除后 commit），保证任何误操作都可从 Git 历史还原。
- 提交信息遵循 Conventional Commits（`feat:` `fix:` `docs:` `style:` `refactor:` `perf:` `test:` `chore:`）。
- 提交前检查 `git status` / `git diff`，只暂存本任务相关文件，绝不提交密钥/凭据。
- 提交后 `git push origin <branch>`，确保远端可还原。

### 4.3 代码修改必须考虑上下文与对接模块
- 修改前先读调用方与被调用方：路由 ↔ service ↔ 数据层 / 进程角色 / 数据流全链都要看，**禁止只见代码块内实现**。
- 至少检查：`app/routes` ↔ `app/services` 的引用、与 `data_sources/data_providers` 的交互、Celery/trading worker 归属、循环依赖。
- 新增/改名函数、接口、DB 字段时，同步检查并更新所有调用点（用 grep/全局搜索验证）。
- 不确定的领域模型先查 `app/services/` 同名文件与 `migrations/` 中对应表结构。

### 4.4 影响部署时必须更新文档
- 改动影响以下任一维度时，**必须**同步更新文档：
  - docker-compose（service/镜像/端口/卷/入口命令/健康检查）
  - 环境变量或配置类（`env.example`、`app/config/*`）
  - 数据库 schema 或迁移
  - 进程角色 / 启动方式 / install.sh / install.ps1
  - API 契约（`docs/api/openapi.yaml`、`docs/agent/agent-openapi.json`）
  - 生产加固或可观测性（`PRODUCTION_HARDENING.md`、`OBSERVABILITY.md`）
- 文档索引以 `docs/README.md` 为准；中英双语文档按项目惯例成对维护（新增英文文档时应补对应 `_CN`，反之亦然，机器可读契约恒为英文）。

## 5. 仓库记忆

- 版本：根 `VERSION` 与 `backend_api_python/VERSION` 必须一致（`scripts/check_version.py` 校验）。
- CI：`.github/workflows/` 有 5 条流水线（basic-ci / docker-publish / mcp-ci / openapi-ci / security-ci）。
- 双 API：人类 API（`/api`，flask-smorest 自动 OpenAPI）与 Agent Gateway（`/api/agent/v1`）。
- 部署模型：5 个 compose 文件（`docker-compose.yml` 核心 + `ghcr` / `build` / `production` / `observability` 覆盖层）。
- 高危文件：凡涉及实盘下单/仓位/网格/撤单/凭据加密/计费/迁移的改动都要格外谨慎并加测试。