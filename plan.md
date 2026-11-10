# llm-radar 完整结构（NAS 产数据 + Cloudflare 只读展示）

## 一、顶层架构图
```
┌──────────────────────── NAS（fnOS，Docker）────────────────────────┐
│  调度：fnOS 任务计划（每天一次）                                     │
│      ↓                                                             │
│  抓取任务（Node/TS）                                                │
│    Fetcher → raw 落盘 → Parser(纯函数) → Store → 本地 SQLite（主）   │
│                                                      ↓             │
│                                         增量变更集 → 同步到 D1       │
│                                                                   │
│  本地数据：raw/ + crawl.db + logs/  ──备份──→ 第二块盘 / 对象存储     │
└──────────────────────────────────┬────────────────────────────────┘
                                   │ D1 HTTP API（唯一上传通道）
                                   ▼
┌──────────────────── Cloudflare（Free，只读）───────────────────────┐
│  D1（同步副本：sources / models / price_history / ...）             │
│      ↓ 同步成功后触发 POST /admin/snapshot                          │
│  KV "latest" 快照                                                  │
│      ↓                                                             │
│  Worker（只读）                                                     │
│    GET /api/models             → KV 快照                           │
│    GET /api/models/:k/history  → D1（带缓存）                       │
│    POST /admin/snapshot        → 读 D1 重建 KV（CF Access 保护）     │
│  Dashboard（Vite+React SPA，Workers Assets 托管，同域免 CORS）       │
└───────────────────────────────────────────────────────────────────┘
```

**一句话职责**：NAS 负责"产数据"（抓取、解析、落库、备份、上传），CF 负责"供数据"（存储副本、只读 API、网页展示）。

## 二、职责边界
| 侧 | 负责 | 明确不负责 |
|----|------|-----------|
| NAS | 抓取、解析、调价判定、下架判定、汇率、本地主库、备份、同步 D1、触发快照 | 对外 HTTP API、网页托管 |
| CF | D1 副本存储、KV 快照、公开只读 API、Dashboard 托管 | 抓取、Cron 调度、配置 CRUD、写业务数据 |

- CF 侧**没有** Cron 抓取、**没有** Admin 模块、**没有**配置编辑。
- CF 侧唯一写入动作：`POST /admin/snapshot` 重建 KV。
- 业务数据的唯一上传通道：**D1 HTTP API**（NAS 直连 Cloudflare API，不经过 Worker）。
- GitHub 仓库只放代码 / schema / 契约 / 种子配置，**不放数据**；部署 Worker 用 `wrangler deploy` 或 GitHub Actions。

## 三、项目目录结构
```
llm-radar/
├── wrangler.jsonc            # CF 绑定：d1 / kv / assets（无 cron）
├── schema.sql                # 唯一 schema 真相，NAS SQLite 与 D1 双侧执行
├── migrations/               # 顺序编号，双侧执行
├── default-sources.json      # 出厂种子配置（运行时不读，仅首次导入用）
├── contracts/                # 冻结契约（先于实现）
│   ├── api.ts                #   /api 响应 schema（Zod）
│   ├── parser.ts             #   Parser 函数签名
│   └── snapshot.ts           #   KV "latest" 快照 schema
├── shared/                   # NAS 与 Worker 共用（平台无关）
│   ├── fetcher.ts            #   全局 fetch 封装（超时/重试/UA）
│   ├── parsers/              #   纯函数 (raw, params) => Item[]
│   └── normalize.ts          #   模型名 / 币种 / 单位归一化
├── src/                      # CF Worker（只读）
│   ├── index.ts              #   Hono 路由挂载
│   ├── api/routes.ts         #   /api/*
│   ├── snapshot.ts           #   POST /admin/snapshot 重建 KV
│   └── repo/readonly.ts      #   D1 只读访问
├── web/                      # Dashboard（Vite + React + TS + Tailwind）
└── nas/                      # ⚠️ NAS 侧实现，技术方案待细化
    └── README.md             #   只约定职责与对 D1 的接口，不预设实现
```

## 四、数据模型（schema.sql）
> 同一份文件在 NAS 的 SQLite 与 CF 的 D1 上各执行一次。
> D1 默认**强制外键**（等价 `PRAGMA foreign_keys=ON`）；本地 SQLite 需在连接建立后手动执行 `PRAGMA foreign_keys=ON;`。
> 因此同步顺序必须父先子后：`sources` → `models` → `model_sources` → `price_history`。

```sql
-- 抓取源配置（后台编辑的唯一对象，唯一事实来源在 NAS 本地库）
CREATE TABLE sources (
  id          INTEGER PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,
  parser      TEXT NOT NULL,               -- 'openai' | 'anthropic' | 'generic_json'
  url         TEXT NOT NULL,
  params      TEXT NOT NULL DEFAULT '{}',  -- JSON，按 parser 用 Zod 校验
  enabled     INTEGER NOT NULL DEFAULT 1,
  last_run_at TEXT,
  last_status TEXT,                        -- 'success' | 'empty' | 'error'
  last_error  TEXT,
  created_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);

-- 模型实体（跨源统一）
CREATE TABLE models (
  id             INTEGER PRIMARY KEY,
  model_key      TEXT NOT NULL UNIQUE,     -- 对外稳定 ID，如 'openai/gpt-4o'
  provider       TEXT NOT NULL,
  model_name     TEXT NOT NULL,            -- 写入前 lowercase + trim 归一化
  context_length INTEGER,
  status         TEXT NOT NULL DEFAULT 'active',  -- active | deprecated | retired
  created_at     TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at     TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')), -- 增量同步标记
  UNIQUE(provider, model_name)
);

-- 模型 × 源：一个模型可能被多个源观测，miss_count 驱动下架判定
CREATE TABLE model_sources (
  model_id      INTEGER NOT NULL REFERENCES models(id),
  source_id     INTEGER NOT NULL REFERENCES sources(id),
  source_key    TEXT NOT NULL,             -- 源里的原始标识，改名时用于追踪
  first_seen_at TEXT NOT NULL,
  last_seen_at  TEXT NOT NULL,
  miss_count    INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (model_id, source_id)
);

-- 价格历史：一个序列 = (model_id, source_id, tier, currency)
-- 口径统一 per_1M tokens；金额是原始币种的整数最小单位（1e-6），不用 REAL 比较
CREATE TABLE price_history (
  id             INTEGER PRIMARY KEY,
  model_id       INTEGER NOT NULL REFERENCES models(id),
  source_id      INTEGER NOT NULL REFERENCES sources(id),
  tier           TEXT NOT NULL DEFAULT 'standard',  -- standard | cached_input | batch
  currency       TEXT NOT NULL,                     -- 'USD' | 'CNY'
  input_price    INTEGER,
  output_price   INTEGER,
  fx_rate_to_usd REAL,                              -- CNY 行必填，写入时快照
  fx_rate_at     TEXT,
  captured_at    TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
CREATE INDEX idx_history ON price_history(model_id, source_id, tier, currency, captured_at DESC);

-- 同一序列同一时刻唯一：同步重推时 ON CONFLICT DO NOTHING 保证幂等
CREATE UNIQUE INDEX idx_price_event
  ON price_history(model_id, source_id, tier, currency, captured_at);

-- 汇率（每天一行，历史曲线用它做可复现换算）
CREATE TABLE fx_rates (
  base       TEXT NOT NULL,
  quote      TEXT NOT NULL,
  as_of      TEXT NOT NULL,
  rate       REAL NOT NULL,
  fetched_at TEXT NOT NULL,
  PRIMARY KEY (base, quote, as_of)
);

-- 抓取运行：NAS 单进程串行执行，不需要跨进程互斥锁
CREATE TABLE crawl_runs (
  id             INTEGER PRIMARY KEY,
  run_key        TEXT NOT NULL UNIQUE,     -- 如 '2026-11-10'
  trigger        TEXT NOT NULL,            -- 'schedule' | 'manual'
  status         TEXT NOT NULL,            -- 'running' | 'success' | 'partial' | 'failed'
  started_at     TEXT NOT NULL,
  heartbeat_at   TEXT NOT NULL,
  finished_at    TEXT,
  sources_total  INTEGER NOT NULL DEFAULT 0,
  sources_ok     INTEGER NOT NULL DEFAULT 0,
  sources_failed INTEGER NOT NULL DEFAULT 0,
  error          TEXT
);

-- 单源运行结果：items_found=0 记为 'empty'，算失败，不参与下架计数
CREATE TABLE source_runs (
  id            INTEGER PRIMARY KEY,
  run_id        INTEGER NOT NULL REFERENCES crawl_runs(id),
  source_id     INTEGER NOT NULL REFERENCES sources(id),
  status        TEXT NOT NULL,              -- 'success' | 'empty' | 'error'
  http_status   INTEGER,
  items_found   INTEGER,
  items_changed INTEGER,
  duration_ms   INTEGER,
  raw_hash      TEXT,
  error         TEXT,
  finished_at   TEXT NOT NULL
);
CREATE INDEX idx_source_runs ON source_runs(source_id, finished_at DESC);

-- 同步检查点：仅 NAS 侧使用（last_sync_at 等）；D1 侧建表但不用
CREATE TABLE sync_state (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

## 五、模块边界规则（核心约束）
```
NAS（唯一写侧）
  Fetcher → Parser → Store → SQLite(主) → Syncer → D1(副)
CF（只读侧）
  /api/* 只读；唯一写动作是 /admin/snapshot 重建 KV
```
- `parsers` 是纯函数 `(raw, params) => Item[]`：不 fetch、不碰 DB，可离线跑 fixture。
- 调价判定只在 store：只比原始币种金额 + tier，换算后的值不参与判定。
- `schema.sql` 是唯一 schema 真相；migration 必须双侧执行（先本地、后远程）。
- **SQLite 是主副本，D1 是同步副本**；D1 上的人工修改一律视为错误，会被下一轮同步覆盖。
- KV 只是可重建的快照缓存，任何时候都可以从 D1 重建。
- 配置（sources）在 NAS 本地库编辑，下一次同步自动带到 D1；CF 侧不提供配置 CRUD。
- 同步必须幂等：`ON CONFLICT` / `INSERT OR IGNORE`，同一批数据重推不产生重复行。
- 同步顺序父先子后（D1 强制外键）：`sources` → `models` → `model_sources` → `price_history` → `fx_rates`。
- CF 侧不跑 Cron、不抓取、不写业务数据。

## 六、数据流
```
① 抓取流（NAS，每天一次）
   调度触发
   → 读 sources(enabled=1)
   → 逐源: Fetcher → raw 落盘 → Parser(纯函数) → Store
       · 按 (model, source, tier, currency) 取上一次价格
       · 原始币种金额变了才追加 price_history
       · upsert models / model_sources，命中则 last_seen_at=now、miss_count=0
   → 写 source_runs，回写 sources.last_*
   → 下架判定（只针对 status='success' 且 items_found>0 的源）
       · 该源未命中的 model_sources：miss_count += 1
       · miss_count >= 3 → models.status='retired'（软删，历史保留）
       · empty / error 的源不动 miss_count，避免一次抓挂就清空目录
   → 每日一次拉 USD/CNY 汇率写 fx_rates
   · 单源失败不阻塞其他源；失败详情落 logs/ 与 source_runs.error

② 同步流（NAS → CF）
   从 SQLite 取本轮变更集
   → D1 HTTP API `{ "batch": [...] }`（幂等 upsert）
   → 成功后 POST /admin/snapshot → Worker 读 D1 重建 KV "latest"
   · 同步失败：指数退避重试；不回滚本地，下一轮按同一变更集补推
   · 快照重建失败不影响 D1 数据，可手动重触发

③ 配置流
   default-sources.json ─[首次导入]→ NAS 本地 sources
   → 编辑（本地库/管理脚本）→ 下一次同步自动到 D1
   → 导出 JSON 备份迁移

④ 查询流
   GET /api/models?currency=USD|CNY      → KV 快照（默认 USD）
   GET /api/models/:model_key/history    → D1（按 captured_at 当日汇率折算）
   Dashboard                             → 同域调用 /api
```

## 七、同步到 D1（关键实现）
### 端点（2026-11 核对官方文档）
```
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/d1/database/{database_id}/query
Authorization: Bearer <API_TOKEN>        # 权限只要 D1:Edit
Content-Type: application/json
```
body 两种形态，**没有独立的 `/batch` 端点**：
```jsonc
// 单条
{ "sql": "SELECT ...", "params": ["a", 1] }

// 多条，同一事务（同步必须用这个）
{ "batch": [
    { "sql": "INSERT ... ON CONFLICT ...", "params": [...] },
    { "sql": "INSERT ... ON CONFLICT ...", "params": [...] }
] }
```
- `sql` 里用分号拼多条时只能带一套 `params`；需要逐条参数化就用 `batch`。
- 需要数组形式的返回（省序列化）可换 `/raw` 端点。
- 注意：D1 查询的执行与结果序列化计入 Worker CPU；NAS 侧不受此限。

### 幂等写法
```sql
-- models
INSERT INTO models (model_key, provider, model_name, context_length, status)
VALUES (?1, ?2, ?3, ?4, ?5)
ON CONFLICT(model_key) DO UPDATE SET
  model_name     = excluded.model_name,
  context_length = excluded.context_length,
  status         = excluded.status,
  updated_at     = strftime('%Y-%m-%dT%H:%M:%fZ','now');

-- model_sources
INSERT INTO model_sources (model_id, source_id, source_key, first_seen_at, last_seen_at, miss_count)
VALUES (?1, ?2, ?3, ?4, ?5, ?6)
ON CONFLICT(model_id, source_id) DO UPDATE SET
  source_key   = excluded.source_key,
  last_seen_at = excluded.last_seen_at,
  miss_count   = excluded.miss_count;

-- price_history（纯追加，靠唯一索引兜底）
INSERT INTO price_history (model_id, source_id, tier, currency, input_price, output_price,
                           fx_rate_to_usd, fx_rate_at, captured_at)
VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9)
ON CONFLICT DO NOTHING;

-- sources
INSERT INTO sources (id, name, parser, url, params, enabled, last_run_at, last_status, last_error)
VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9)
ON CONFLICT(id) DO UPDATE SET
  name = excluded.name, parser = excluded.parser, url = excluded.url,
  params = excluded.params, enabled = excluded.enabled,
  last_run_at = excluded.last_run_at, last_status = excluded.last_status,
  last_error = excluded.last_error,
  updated_at = strftime('%Y-%m-%dT%H:%M:%fZ','now');
```

### 增量与检查点
- 以本轮 `run_id` 产生的变更集为准，不整表推。
- `sync_state.last_sync_at` 记录成功时间；失败重试按同一变更集重推（幂等，安全）。
- 每批控制在几百条语句内，避免单请求过大；必要时按表分批。

### wrangler 与 HTTP API 的分工
| 用途 | 工具 |
|------|------|
| 建库 `wrangler d1 create` | wrangler |
| 建表 / migration `wrangler d1 execute --remote --file=schema.sql` | wrangler |
| 一次性灌库 / 紧急修数据 | wrangler |
| 本地开发库 `--local` | wrangler |
| **每日增量同步（生产路径）** | **D1 HTTP API** |

wrangler 是给人用的，HTTP API 是给程序用的；后端是同一套 D1。

### Token 与密钥
- API Token 只勾 `Account | D1 | Edit`，限定到本账号，按需设过期时间。
- 存 NAS 的 `.env`，权限 600，`.env` 必须进 `.gitignore`。
- 不给 Workers Scripts / Account Settings 权限。

## 八、本地数据布局（NAS）
```
<data 目录>/
├── raw/YYYY-MM-DD/<source>.json     # 原始响应，不可再生，必须留
├── crawl.db                         # 处理后的主副本（单文件 SQLite）
├── logs/YYYY-MM-DD.log              # 运行日志
└── backup/                          # 可选：每日 VACUUM INTO 快照
```
- `raw/` 用于：parser 改 bug 后离线重跑、D1 灾难重建、生成 golden fixture。
- `crawl.db` 是主副本，**SQLite 文件本身就是备份**，不需要额外"导出格式"。
- 备份 = `raw/` + `crawl.db` 一起复制到第二块盘 / 外接盘 / 对象存储。
- 建议每日任务结束后执行 `VACUUM INTO` 生成当日快照，避免备份到写入中的文件。
- 具体调度方式、容器编排、日志轮转、备份目标由 NAS 侧方案细化。

## 九、Cloudflare Free 额度预算
> 数据来源：Workers Limits、D1 Limits（2026-10 更新）。

| 限制 | Free 额度 | 本设计如何满足 |
|------|-----------|----------------|
| Worker 数量 | 100 / 账号 | 只用 1 个 |
| Cron Triggers | 5 / **账号** | CF 不跑 Cron；最多留给可选 watchdog 1 个 |
| 每日请求 | 100,000 / 账号 | 公开 API 量级远低于 |
| CPU / 次调用 | 10 ms | `/api/models` 读 KV，不读 D1 大结果集 |
| 子请求 / 次调用 | 50 | 读侧只有 1 次 KV 或 1~2 次 D1 |
| D1 查询 / 次调用 | 50（`batch()` 算 1） | history 单条查询 + 索引 |
| D1 单库 / 账号存储 | 500 MB / 5 GB | 价格历史量级远低于 |
| D1 库数量 | 10 / 账号 | 只用 1 个 |
| Cron 墙钟 | 15 min | 不使用 |
| 同时出站连接 | 6 | 读侧无并发抓取 |

关键约束：**D1 查询执行与结果序列化计入 Worker 的 10 ms CPU**，所以公开列表接口必须读 KV 快照，不能每次扫 D1。

## 十、关键决策一览
| 决策点 | 结论 |
|--------|------|
| 部署形态 | NAS（fnOS Docker）产数据 + CF Free 单 Worker 只读展示 |
| 主副本 | NAS 的 `crawl.db`（SQLite）；D1 是同步副本 |
| 上传通道 | D1 HTTP API（生产）；wrangler 仅建库 / migration / 一次性灌库 |
| 本地备份 | `raw/` + `crawl.db` 复制到第二位置；SQLite 文件即备份格式 |
| JSON 文件 | 种子文件放仓库根目录，运行时不读 |
| 解析器 | 代码层 `shared/parsers/`；新 JSON API 厂商用 `generic_json` 加源不写代码 |
| 历史表 | 只记录调价事件，曲线干净；唯一索引保证同步幂等 |
| 计价口径 | 统一 per_1M tokens，原生币种入库（USD/CNY），展示默认 USD、可切 CNY |
| 汇率 | NAS 每日抓一次存 `fx_rates`；按 `captured_at` 当日汇率折算，不用当前汇率重算历史 |
| 调价判定 | 只比原始币种金额 + tier；用换算值比较会让汇率波动刷出假调价 |
| 模型下架 | 软删：连续 3 轮在成功源中未命中 → `status='retired'`，默认隐藏、历史保留 |
| 多源冲突 | 每个 `(model, source)` 独立序列，写入期不合并；跨源比价在查询期算 |
| 抓取互斥 | NAS 单进程串行 + SQLite 事务；不再需要 CF 侧的部分唯一索引锁 |
| parser 测试 | 每个 parser 配 golden fixture，CI 失败即阻断部署 |
| 零结果 | `items_found=0` 记为 `empty`（失败），不参与下架计数，同时告警 |
| CF 侧写入口 | 仅 `POST /admin/snapshot`（CF Access 保护），无 CRUD |
| 管理鉴权 | CF Access 套 `/admin/*`，零代码 |
| Dashboard | Vite+React SPA，Assets 托管，同域 |
| R2 | 砍掉，备份走本地 SQLite + raw |
| 未来自部署 | NAS 已是主副本：停掉同步、把读 API 换成本地服务即可，代码复用 |
| 告警 | NAS 任务结束打心跳（Healthchecks.io 等），超时未收到即告警 |

## 十一、实施顺序
```
⓪ 冻结契约：/api JSON schema、parser 函数签名、snapshot schema、fixture 目录约定
① NAS 本地跑通：schema.sql → SQLite；1~2 个 parser + fixture；抓取 → raw + crawl.db
   （此阶段不连 CF，验证解析、调价判定、下架判定）
② CF 只读侧：wrangler d1 create + schema.sql 远程；Worker /api/* + /admin/snapshot + KV
   （用 wrangler 灌几条假数据，curl 能拿到 JSON）
③ 打通同步：d1Http 驱动 + 增量 batch upsert + 幂等 + 重试；端到端跑通
④ Dashboard + CF Access（/admin/snapshot）+ 备份 + 告警
⑤ deploy + 连续观察一周（额度、错误率、同步耗时）
```
每一阶段都可独立验证、可回退；① 结束时已经有一个可用的本地系统。

## 十二、待细化（NAS 侧）
- [ ] 调度方式：fnOS 任务计划 / 宿主 crontab / 容器内 supercronic
- [ ] Docker 镜像与 compose 编排（Node 版本、挂载、时区 `Asia/Shanghai`）
- [ ] 同步失败的重试策略与告警通道
- [ ] 本地备份目标与轮转（第二块盘 / 外接 / B2）
- [ ] `raw/` 保留期限与清理策略
- [ ] 汇率数据源与容错
- [ ] 单源超时、限速、User-Agent 策略
- [ ] 配置编辑方式（直接 SQL / 管理脚本 / 本地小 UI）
