# QuantDinger 安全审计报告（2026-09-13）

> 归档类型：只读安全审计产出。**本次仅归档，不包含代码修复。**
> 审计方法：4 路并行只读代码审查（认证授权 / 注入输入 / 密钥配置部署 / MCP 交易支付）+ 关键证据人工复核。
> 版本基线：`VERSION = 5.0.17`；HEAD 提交：`df9ce9f`。
> 后续修复完成后，请在本文件末尾追加「修复记录」章节，逐条标注处置状态与提交号。

## 1. 总体评价

项目安全基线**良好**：SQL 全参数化、IDOR 普遍按 `user_id` 作用域、TOTP 常数时间比较 + 防重放、JWT 每次校验数据库 token 版本、无真实密钥入库、生产 overlay 加固到位。

但存在 **1 个 Critical（默认口令）与 5 个 High**，其中部分在「开放端口 / 上云」部署时会升级为可利用漏洞。默认 loopback 部署下多数问题影响有限。

## 2. 漏洞清单（按严重级别）

### Critical

- **C-1 默认管理员口令 `quantdinger / 123456`，且登录不拦截**
  - `backend_api_python/app/config/settings.py:55`：`ADMIN_PASSWORD = os.getenv('ADMIN_PASSWORD', '123456')`
  - `env.example:55-58` 出厂即带默认口令；`backend_api_python/docker-entrypoint.sh:12-28` 无 `.env` 时静默复制 `env.example` 把默认口令带入运行环境
  - `backend_api_python/app/services/user_service.py:64-65` 引导默认账号
  - 虽有「首次登录强制改密」提示（`auth.py:51-57`、`user_service.py:174-192`），但登录端点**仍签发令牌**；部署漏配或沿用默认口令 = 已知管理员账户接管
  - 建议：默认口令 fail-fast 拒绝启动；未改密用户禁止登录（`password_changed_at` 为空则拒绝）；`login-code` 自动建号路径同样复查

### High

- **H-1 SECRET_KEY 兼容下限仅 10 字节（HS256 伪造面）**
  - `backend_api_python/app/config/settings.py:10`：`_MIN_SECRET_KEY_BYTES = 10`；10-31 字节仅警告
  - `backend_api_python/app/utils/auth.py:31-40`：`_RECOMMENDED_JWT_SECRET_BYTES = 32`，仅建议
  - HS256 用 `Config.SECRET_KEY` 签名/验签（`auth.py:79-83 / 101-104`）；低熵密钥可离线爆破 → 认证全面绕过（含 admin 角色）
  - 建议：新部署下限提升至 32 字节并 fail-fast；移除对不安全长度告警的抑制

- **H-2 凭据加密密钥 = SHA256(单一密钥)，无独立密钥层级与轮换**
  - `backend_api_python/app/utils/credential_crypto.py:34-45`：Fernet 密钥 = `base64(sha256(secret))`，确定性、无 KMS；`CREDENTIAL_ENCRYPTION_KEY` 为空时回退 `SECRET_KEY`
  - 保护对象：Alpaca/IBKR 券商密钥、TOTP MFA 种子、桌面券商策略数据
  - SECRET_KEY 泄露或低熵（H-1）→ 全部第三方凭据可离线解密
  - 建议：独立存储的数据加密密钥；强制 32+ 字节；轮换时显式重加密遗留密文

- **H-3 用户策略/指标 Python 代码在 Web worker 进程内 `exec()`**
  - `backend_api_python/app/services/strategy_v2/contract.py:195-233`：`compile_strategy_v2` → `safe_exec_with_validation` → `exec(code, ...)`
  - 请求链：`app/routes/script_source_routes.py:192,311`（compile）、`routes/strategy.py:263`、`routes/backtest_center.py:173`、`services/strategy_v2/deployment.py:25`、`runtime.py:1712/2241`、`community_service.py:2657`
  - 指标代码另有进程内计算路径（`app/services/indicator_params.py:300-309`）
  - 现缓解较强：`safe_exec.py` 白名单 builtins + 白名单 import（numpy/pandas/math）+ regex+AST 双重静态分析（屏蔽 os/sys/subprocess/getattr/dunder 链/pandas·numpy IO）+ `_SafeModuleProxy` + 错误脱敏 + `python -I` 子进程隔离已存在但仅 isolated 路径使用
  - 风险：Python 沙箱（尤其开放 pandas/numpy）存在已知逃逸面；一旦逃逸 = 持有 DB/交易所凭据的进程内 RCE
  - 建议：统一迁移到 `safe_exec_isolated`（JSON-only、`python -I`、clean env）边界；外层加 OS 级沙箱（seccomp/Landlock、非 root、容器内存/CPU 上限）

- **H-4 生产 overlay 只读挂载 `.env`，entrypoint 静默回退内存随机密钥**
  - `docker-compose.production.yml:3-4,13`：`user: 10001:10001`、`read_only: true`、`.env` 以 `:ro` 挂载
  - `backend_api_python/docker-entrypoint.sh:36-47`（SECRET_KEY 缺失时写入失败静默回退内存随机值）、`:92-102`（CREDENTIAL_ENCRYPTION_KEY 同）
  - 多副本各自持不同密钥 → JWT 跨副本验签失败、已加密凭据解密失败、每次重启密钥轮换；GHCR 零仓库部署 `backend.env` 被 docker 创建为目录遮蔽时同样落入此分支
  - 建议：生产 overlay 启动前强制宿主已显式设置 `SECRET_KEY` 与 `CREDENTIAL_ENCRYPTION_KEY`；entrypoint 检测到 in-memory 模式直接 `exit 1` fail-fast

- **H-5 OAuth 令牌/授权码进入 URL 与访问日志，provider token 明文入库**
  - `backend_api_python/app/routes/auth.py:1104`（Google 回调 `oauth_token=token`）、`:1201`（GitHub 同）、`:1060-1110 / :1157-1207`（错误路径带 `oauth_error`）
  - `backend_api_python/gunicorn_config.py:30-31`：`accesslog = "-"` 默认格式含完整查询串 → OAuth code/state 进容器日志
  - JWT 经 `Location` 回落前端：hash 路由进浏览器历史（`auth_session.py:14-55`）；`FRONTEND_URL` 为真实路径模式时令牌进查询串 → 泄露给 Referer 与访问日志；JWT 有效期 7 天
  - `backend_api_python/app/services/oauth_service.py:492-498,531-538,591-598`：Google/GitHub `access_token`/`refresh_token` 明文写入 `qd_oauth_links`（migrations/init.sql:285-286）
  - 建议：令牌改经一次性授权码或 HttpOnly Secure cookie；gunicorn 自定义访问日志格式丢弃查询串；OAuth provider token 用 `credential_crypto` 加密

### Medium

- **M-1 MFA(TOTP) 默认关闭**
  - `backend_api_python/app/services/mfa_service.py:60-63`：`MFA_ENABLED` 默认 False、`MFA_RISK_LOGIN_ONLY` 默认 True → 默认全账号仅口令，叠加 C-1

- **M-2 邮箱验证码：非 CSPRNG 生成 + 明文存储 + 明文比较**
  - `backend_api_python/app/services/email_service.py:67-69`：`random.choices(string.digits, k=6)`（Mersenne Twister）
  - `migrations/init.sql:242`：`code VARCHAR(10) NOT NULL` 明文存储；`email_service.py:172` 明文比较
  - 门控面：`/reset-password`（auth.py:884）、`/login-code` 自动建号（auth.py:371-460）、改密（auth.py:948）
  - 缓解：5 次尝试锁 30 分钟、10 分钟过期、60s 重发节流、10/小时/IP 上限
  - 建议：`secrets` 生成，存/比 SHA-256

- **M-3 Webhook 出站 SSRF（无私网地址封禁，响应体部分回显）**
  - `backend_api_python/app/services/signal_notifier.py:704-852`：仅校验 `http://`/`https://` 前缀（741-742），无私网/保留/元数据 IP（127.0.0.0/8、10/8、172.16/12、192.168/16、169.254.169.254、::1）封禁
  - 用户可在通知 target 提交任意 webhook_url（`indicator_signal_alerts.py:946-953`、`portfolio_monitor.py:1115-1278`）
  - `signal_notifier.py:143` `shorten(text,300)` 把非 2xx 响应体前 300 字符带回 result 消息 → 内网探测
  - 建议：出站前解析 URL，拒绝私网/保留/元数据 IP；域名解析后再校验 IP（防 DNS 重绑定）；限制响应回显

- **M-4 策略编译无内存上限（OOM DoS）**
  - `backend_api_python/app/utils/safe_exec.py:477-487`：RLIMIT_AS 仅 `SAFE_EXEC_ENABLE_RLIMIT=true`（默认 false）且非 Windows 生效；编译仅 10s CPU 超时（`contract.py:233`），无内存上限
  - `x = [0.0] * (10**9)` 类代码可耗尽 API worker 内存触发 OOM
  - 建议：默认开启 RLIMIT 或强制子进程隔离 + 容器内存配额；编译请求限流

- **M-5 PostgreSQL 连接不支持 TLS**
  - `backend_api_python/app/utils/db_postgres.py`：未处理 `sslmode`；`DATABASE_URL` 的 `?sslmode=require` 会被误并入 dbname 导致解析失败（等效静默禁用 TLS）
  - 跨网络部署时口令/持仓/策略代码可被嗅探
  - 建议：支持 `sslmode` 并默认 prefer/require；解析 URL 先剥离查询串

- **M-6 MCP 网关默认零鉴权 + 明文 HTTP**
  - `docker-compose.yml:274-296`：`QUANTDINGER_MCP_HOST=0.0.0.0`、`QUANTDINGER_MCP_AUTH_TOKEN` 默认空、`QUANTDINGER_MCP_ALLOW_HTTP` 默认 true
  - MCP 服务端自身有非回环拒绝启动/令牌长度与相异性检查（`mcp_server/src/quantdinger_mcp/server.py:1124-1147`），但 `QUANTDINGER_MCP_ALLOW_INSECURE_HTTP=true` 可允许完全无登录 token 的远程连接 → 未认证远程客户端以预设 token 触达全部 58 个工具
  - 建议：token 默认随机生成；ALLOW_HTTP 默认 false；移除 INSECURE 逃生舱

- **M-7 Agent 实盘下单/停止/取消的 confirm 仅在 MCP 客户端，服务端不强制**
  - `backend_api_python/app/routes/agent_v1/quick_trade.py:404`（place_order，`_ORDER_FIELDS` allowlist 无 confirm 字段）、`runtime.py:105`（stop_strategy）、`jobs.py:41`（cancel_job）
  - 对比：kill_switch（quick_trade.py:520-521）、restore（strategy_sources.py:219-220）、alert run（notifications.py:85-86）均服务端强制 confirm
  - 持 live token 直接调 REST 即可无确认实盘下单/停策略
  - 建议：服务端对 place_order（实盘名义金额>0 时）、stop_strategy、cancel_job 强制 `confirm: true`

- **M-8 市场购买积分并发丢更新（透支）**
  - `backend_api_python/app/services/community_service.py:1063` `purchase_indicator`：余额读（get_user_credits:1114/1140）后写绝对值（`UPDATE qd_users SET credits=?`:1123-1126,1142-1145），无行锁 / 无 `WHERE credits>=?` / 无 rowcount 检查
  - 两笔不同指标并发购买可同时通过余额检查，最终余额只扣一次（余额 1500，两笔并发 1000 均成功，最终 500）
  - 建议：事务内 `SELECT ... FOR UPDATE` 后 `UPDATE credits = credits - ? WHERE id=? AND credits>=?` 并验 rowcount；卖家同理

- **M-9 验证码限频 DB 异常时 fail-open**
  - `backend_api_python/app/services/security_service.py:411-413`：`except Exception: return True, 'allowed'` → DB 抖动时验证码轰炸/暴力尝试窗口打开（对比 Turnstile 是 fail-closed）
  - 建议：改为有限降级（强制冷却/拒绝）并告警

### Low

- **L-1** JWT 有效期 7 天（`auth.py:72`）；SECRET_KEY 告警被抑制（`auth.py:44` `filterwarnings ignore`）
- **L-2** `.gitignore` 未覆盖非 `.local` 的 env 变体（`.env.production`、`.env.test`、`env.testnet` 等可被误提交）
- **L-3** 密钥生成脚本回显明文（`scripts/generate-secret-key.sh:17,30`、`.ps1:24,38`）；遗留单用户模式明文口令比较（`auth.py:331-349`）
- **L-4** 凭据加密用 SHA256 非 KDF（`credential_crypto.py:34-36`），主密钥熵低时可离线穷举
- **L-5** OAuth 日志记录 provider 原始响应体（`oauth_service.py:280,294`）
- **L-6** `.env` 写入转义不覆盖 `$` 与反引号（`app/services/settings/env_file.py:81-91`；admin-only + schema 白名单缓解）
- **L-7** 指标参数接口 ID 隔离缺口（`app/services/indicator_params.py:363` 无 user_id 条件，仅存在性 oracle，无代码泄露）
- **L-8** `/metrics` 未认证、`/api/health/ready` 泄露内部状态（`openapi/routes/health.py:115-123,71-81`）
- **L-9** 无安全响应头（HSTS/CSP/X-Frame-Options）；前端代码不在本仓库，需在反代层配置

### Info / 已验证安全（无需处理）

- SQL 注入：`db.py` / `db_postgres.py` / 各 services 全量静态分片 + 参数化，无字符串拼接注入
- IDOR：dashboard.py:658/675/779/788、analysis_memory.py:432、ai_copilot_store.py:250/340/398、agent_v1/trading_data.py、billing/usdt 订单均按 `user_id` 作用域
- CORS：`app/__init__.py:71-92` 显式 origins、`supports_credentials=False`
- OAuth state：一次性消费 + 过期 + `qd_oauth_states` 共享表，回调拒绝 `urlencoded:` / `//` 遍历；前端跳转白名单（无开放重定向）
- JWT：7 天有效期但 DB 复核 user_id/token_version 权威覆盖伪造角色声明；单客户端 token_version 踢下线；登录失败按 IP/账户记录锁定
- 支付幂等：USDT 唯一索引 + 随机金额后缀（10001 槽位）、Stripe `compare_digest` + 300s 窗口、`purchase_membership` `ON CONFLICT DO NOTHING` 只发放一次
- 券商/2FA 凭据加密存储：`exchange_execution.py` / `mfa_service.py` 走 `decrypt_credential_blob`
- 生产 overlay：非 root UID 10001、`cap_drop ALL`、只读 rootfs、tmpfs noexec/nosuid/nodev
- TOTP：`mfa_service.py:305-359` 常数时间比较 + `last_used_counter` 防重放 + 恢复码哈希
- 图片附件：仅 base64 data URL（MIME/大小白名单），不落盘
- 历史高危（2026-07 JWT 绕过、2026-08 执行隔离、2026-09 代理头信任）均已在 SECURITY.md 留档并修复

## 3. 修复优先级

| 优先级 | 事项 |
|---|---|
| P0 | C-1 默认口令 fail-fast + 登录拦截；H-4 生产密钥持久化 fail-fast；H-5 access log 去查询串 + OAuth token 加密 |
| P1 | H-1/H-2 密钥下限 32 字节 + 独立 KDF；H-3 统一子进程隔离沙箱 + 内存限制；M-7 服务端强制 confirm |
| P2 | M-3 SSRF、M-5 PG TLS、M-6 MCP 默认强化、M-8 积分原子扣减、M-9 fail-closed |
| P3 | 其余 Low 项与文档清单完善 |

## 4. 修复记录（待后续补齐）

| 严重级 | 编号 | 处置状态 | 提交号 |
|---|---|---|---|
|  |  | 未开始 |  |