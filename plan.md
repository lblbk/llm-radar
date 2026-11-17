# llm-radar 实施方案 v2

NAS 上的 Docker 容器持续运行 Python 服务，由容器内调度器定时抓取；本地 SQLite 保存主数据。抓取结果同步到 Cloudflare D1，Worker 提供公开 API，React 展示站点读取公开数据。后续可在 NAS 服务中增加本地配置界面。镜像由 GitHub Actions 发布到 GHCR，NAS 部署固定版本。

## 一、架构与职责

```text
GitHub：源码 → Actions 构建镜像 → GHCR → NAS 拉取固定版本

NAS（Docker 常驻服务）
  → 内置调度器按计划触发抓取，也可手动触发
  → Fetcher → raw 文件 → Parser → 校验 → SQLite
  → 同步待发送数据到 Cloudflare D1；失败时保留队列并重试
  → 后续增加本地配置 API / Web 界面

浏览器 → Workers Assets：公开 React 页面
       → Worker API → D1：当前报价和历史记录

SQLite + 本地来源配置 + 待同步队列 → 定期备份到 NAS 的备份位置
```

| 部署侧 | 职责 |
|--------|------|
| GitHub / Actions / GHCR | 源码、测试、多架构镜像构建与分发，不运行生产抓取任务 |
| NAS / Python | 常驻服务、定时抓取、SQLite 主库、待同步重试；后续提供本地配置界面 |
| Worker / TypeScript | 公开查询 API 和静态资源路由 |
| D1 | 网站使用的公开数据副本和价格历史 |
| React | 公开模型列表、来源选择、币种切换和历史曲线 |

- CF 不抓取、不调度业务任务、不编辑源配置，不保存源站认证信息。
- NAS 是配置与运行状态的主库；D1 只保存网站需要的公开数据。
- Worker 直接查询 D1；如后续读取量需要缓存，再增加缓存层。
- NAS 上的配置界面是后续功能；源站密钥和抓取配置留在 NAS，不发送到 Cloudflare。
- Git 不存运行数据、密钥、本地数据库或真实响应；测试只提交必要的脱敏样本。

## 二、技术栈与目录

Python 服务是独立的 `llm-radar` Python distribution，导入包名为 `llm_radar`，CLI 命令为 `llm-radar`，位于 `apps/llm-radar/` 并采用标准 `src/llm_radar/` 布局。它使用 httpx、Pydantic、标准库 sqlite3 / decimal、HTML 解析库和 pytest。Docker 启动常驻服务，服务内调度器定时触发抓取；不需要 NAS 主机再设置 Cron。首版不引入 Redis 或 Celery。

```text
llm-radar/
├── README.md
├── .gitignore
├── .env.example                   # 本地/Compose 环境变量样例，不含密钥
├── .dockerignore                  # 根目录是 Docker build context
├── compose.yaml                   # 部署编排，显式指向 apps/llm-radar/Dockerfile
├── .github/workflows/
│   ├── ci.yml                     # PR / 默认分支测试，不发布 PR 镜像
│   └── publish-image.yml          # Release 发布 GHCR 镜像
├── apps/
│   ├── llm-radar/                 # Python distribution: llm-radar
│   │   ├── pyproject.toml
│   │   ├── uv.lock
│   │   ├── Dockerfile
│   │   ├── config/default-sources.json
│   │   ├── src/llm_radar/
│   │   │   ├── cli.py
│   │   │   ├── config.py
│   │   │   ├── fetcher.py
│   │   │   ├── parsers/
│   │   │   ├── normalize.py
│   │   │   ├── store.py
│   │   │   ├── fx.py
│   │   │   ├── syncer.py
│   │   │   ├── backup.py
│   │   │   ├── scheduler.py
│   │   │   └── service.py          # 常驻进程；后续可承载本地配置 API
│   │   └── tests/fixtures/
│   ├── shared/                    # 跨应用复用的契约与数据库代码，不单独部署
│   │   ├── contracts/             # Python / TS 共用 JSON Schema 与正反例
│   │   │   ├── item.schema.json
│   │   │   ├── parser-result.schema.json
│   │   │   ├── models.schema.json
│   │   │   ├── api.schema.json
│   │   │   └── examples/
│   │   └── database/              # 数据库迁移与 D1/SQLite 结构工具
│   │       └── migrations/
│   │           ├── shared/        # SQLite、D1 公共表迁移
│   │           └── local/         # 仅 SQLite：配置、运行、outbox
│   ├── worker/                    # Cloudflare Worker，独立 TypeScript app
│   │   ├── package.json
│   │   ├── package-lock.json
│   │   ├── wrangler.jsonc         # D1 / Assets；NAS 负责抓取调度
│   │   └── src/
│   │       ├── index.ts           # Hono 路由入口
│   │       ├── api/routes.ts
│   │       └── repo/readonly.ts
│   ├── web/                       # Cloudflare 上的公开 Vite / React 页面
│       ├── package.json
│       ├── package-lock.json
│       ├── index.html
│       ├── vite.config.ts
│       └── src/
│   └── admin-web/                 # 后续增加：NAS 局域网配置界面，不部署到 Cloudflare
```

- 根目录只放仓库级文档、环境/构建/部署配置；应用和共享代码统一放在 `apps/`，共享代码集中在 `apps/shared/`。
- `apps/llm-radar/` 是 Python 项目目录，名称与仓库及 distribution 一致；distribution 名是 `llm-radar`，源码导入名是 `llm_radar`。
- `apps/llm-radar/pyproject.toml` 声明 `[project] name = "llm-radar"`、从 `src/` 发现 `llm_radar` 包，并注册 `llm-radar = "llm_radar.cli:main"` console script；`uv.lock` 与该 Python 项目放在同一目录。
- `apps/worker/` 与 `apps/web/` 是独立 Node 项目，各自管理 `package.json` 和依赖；Worker 的 Assets 配置指向 `../web/dist`。CI 按应用目录安装依赖，镜像构建仍以仓库根目录为 context。
- Worker 使用 Hono 处理公开 `/api/*` 路由，通过 D1 binding 读取数据。NAS 配置界面属于后续本地功能，不经过 Cloudflare Worker。
- 取消跨语言的 shared/fetcher.py 和 TS parser；Worker 不需要抓取实现。
- 跨语言共享 JSON Schema、示例和公共迁移；Python/TS 分别校验，CI 验证同一批样本。
- `apps/shared/database/migrations/` 是表结构事实来源，不再同时手写 schema.sql。SQLite 和 D1 分别记录迁移版本。
- 公共迁移先验证 SQLite 和 D1 开发库；上线先做向后兼容的远端扩展，再升级 NAS/Worker，最后移除旧字段。迁移失败禁止同步，不自动删库重建。

### Python 抓取流程与来源适配器

常驻服务和手动 CLI 共用同一套抓取流程。服务内调度器按设定时间触发抓取；每个启用的来源独立处理，来源失败时继续处理其他来源。

1. 创建 `crawl_runs`，按来源配置读取 `sources.enabled`、`priority` 和对应的 `source_configs`。配置中的 `parser` 选择已注册的来源适配器；适配器定义请求构造方式、分页策略及响应解析规则，URL、非敏感参数和密钥引用保存在 NAS 本地。
2. Fetcher 通过 httpx 发出请求，按适配器的分页规则逐页抓取并记录页序。优先使用公开 JSON/API；没有稳定 API 时再解析 HTML。首版串行执行，同域请求间隔、超时、重试和响应大小遵循第十节限制。确实依赖 JavaScript 的来源才使用可选浏览器镜像。
3. 每页响应按运行、来源、尝试和页序压缩写入 `/data/raw/`，同时生成 manifest，记录状态码、内容类型、抓取时间、hash、页序和适配器版本。先完整写临时文件再原子落盘；密钥与认证 headers 不进入 manifest、日志或远端数据。
4. 收齐页面后调用纯函数 `parse(raw_pages, params) -> ParseResult`。Parser 不联网、不访问数据库，将来源字段转换为统一记录，并返回模型、报价、覆盖范围、分页完整性、错误和警告。金额从原始十进制文本用 Decimal 解析，再按公共模型规定的单位转换；不先转 float。
5. Store 校验整个来源结果：必需字段有效、source_key 和报价档位无冲突、币种/单位受支持、页面完整且覆盖声明成立。`source_runs` 记录每个来源的 `success`、`empty`、`incomplete` 或 `error` 结果；只有通过校验的来源数据才参与本轮业务更新。失败或不完整的来源保留旧数据并标记 stale，不能增加模型缺失计数。空目录单独记为 `empty`，不据此确认下架。
6. 网络请求和解析均在 SQLite 写事务外完成。通过校验后，Store 在单一本地事务内应用来源结果、生成价格历史，并把待上传的公开记录放入同步队列；事务规则见第五节。raw 文件供解析回放和故障定位使用，不上传 Cloudflare。

新增来源时，先确定公开 source 元数据、URL、适配器和覆盖范围；若现有适配器不能表达其请求或响应格式，再新增来源解析器。提交脱敏后的真实响应 fixture，覆盖分页结束、空结果、字段缺失和价格单位，并用 fixture 验证解析与完整性判断。实际首批来源及其 endpoint/字段映射尚未在本方案中列出，开始实现前需要确定并写入 `apps/llm-radar/config/default-sources.json`。

## 三、身份、金额与定价维度

1. 源、模型、报价、价格事件和运行 ID 均由 NAS 生成 UUID 的 32 位小写十六进制字符串，两边原样保存。D1 不自行分配实体 ID。
2. 模型公开 URL 使用 models.id，不把含斜杠的厂商名称当路径参数。原始名称保留大小写；canonical_name 由厂商适配器确定，不能统一 lowercase 后猜测别名。
3. provider 使用固定厂商标识，展示名另存。跨源别名通过显式映射归并，没有把握的不自动合并。
4. 一个报价序列由模型、来源、模式（standard / batch）、上下文下限/上限、原币种唯一确定。上下文采用 `[min, max)`，max=0 表示无上限；重叠或不支持的档位拒绝入库。
5. 输入、输出、缓存读取、缓存写入分别存金额；缺失为 NULL，免费为 0。音频、图片、按请求收费等不强行换算 token，第一版标为不支持并告警。
6. 金额统一为“每百万 token 的原币金额 × 1,000,000”的整数。例如每百万 token 2.5 USD 存 2500000。Python 用 Decimal 从原始十进制字符串转换，禁止先转 float；超出精度或整数范围时拒绝静默舍入。
7. JSON 中金额用十进制整数字符串，避免 JS 整数精度丢失。格式化、换算和显示舍入不参与调价判定。
8. 业务时间由 NAS 生成 UTC ISO 8601 毫秒值，远端原样保存；NAS 调度按北京时间，汇率日期按 UTC。
9. 每个价格变化生成一个稳定 ID 和抓取时间；相同数据重试上传时使用原 ID，避免重复历史记录。

## 四、公共数据模型

以下是首版公共数据表的设计基线。SQLite 每个连接启用 `PRAGMA foreign_keys=ON`。

```sql
CREATE TABLE sources (
  id TEXT PRIMARY KEY NOT NULL,
  name TEXT NOT NULL UNIQUE,
  public_url TEXT,
  priority INTEGER NOT NULL DEFAULT 100,
  enabled INTEGER NOT NULL CHECK(enabled IN (0, 1)),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE models (
  id TEXT PRIMARY KEY NOT NULL,
  provider TEXT NOT NULL,
  canonical_name TEXT NOT NULL,
  display_name TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  UNIQUE(provider, canonical_name)
);

CREATE TABLE offers (
  id TEXT PRIMARY KEY NOT NULL,
  model_id TEXT NOT NULL REFERENCES models(id),
  source_id TEXT NOT NULL REFERENCES sources(id),
  mode TEXT NOT NULL CHECK(mode IN ('standard', 'batch')),
  context_min INTEGER NOT NULL DEFAULT 0 CHECK(context_min >= 0),
  context_max INTEGER NOT NULL DEFAULT 0,
  currency TEXT NOT NULL CHECK(currency IN ('USD', 'CNY')),
  status TEXT NOT NULL CHECK(status IN ('active', 'stale', 'suspect_missing', 'unavailable_confirmed')),
  last_seen_at TEXT NOT NULL,
  created_at TEXT NOT NULL,
  CHECK(context_max = 0 OR context_max > context_min),
  UNIQUE(model_id, source_id, mode, context_min, context_max, currency)
);

-- 不可变日汇率；方向固定为 1 USD = usd_cny CNY。
CREATE TABLE fx_rates (
  id TEXT PRIMARY KEY NOT NULL,
  as_of TEXT NOT NULL UNIQUE,
  usd_cny TEXT NOT NULL,
  provider TEXT NOT NULL,
  fetched_at TEXT NOT NULL
);

-- 只记录实际价格变化；当前报价由最新事件确定。
CREATE TABLE price_history (
  id TEXT PRIMARY KEY NOT NULL,
  offer_id TEXT NOT NULL REFERENCES offers(id),
  input_price INTEGER CHECK(input_price >= 0),
  output_price INTEGER CHECK(output_price >= 0),
  cached_input_price INTEGER CHECK(cached_input_price >= 0),
  cache_write_price INTEGER CHECK(cache_write_price >= 0),
  fx_rate_id TEXT REFERENCES fx_rates(id),
  captured_at TEXT NOT NULL,
  CHECK(input_price IS NOT NULL OR output_price IS NOT NULL
     OR cached_input_price IS NOT NULL OR cache_write_price IS NOT NULL)
);
CREATE INDEX idx_price_history
  ON price_history(offer_id, captured_at, id);
```

- sources 只含公开元信息。抓取 URL、headers、参数、secret 引用放 NAS 专用配置表；认证参数、错误堆栈、原始响应不上传。
- offers 定价维度不变；换币种/档位创建新报价序列。当前价格取该报价最新的价格历史记录；网站展示 active 报价，并标注 stale 报价。
- Worker 直接读取 D1 当前数据；模型、来源或报价信息更新后，下一次同步即可反映在网站上。
- 不删除历史实体来表达下架。第一版不支持历史原地修订，确需修订时另行升级可追溯协议。

## 五、NAS 专用模型与事务

以下表只存在于本地 SQLite，CF 不建空表占位。

```sql
CREATE TABLE source_configs (
  source_id TEXT PRIMARY KEY NOT NULL REFERENCES sources(id),
  parser TEXT NOT NULL,
  url TEXT NOT NULL,
  params_json TEXT NOT NULL CHECK(json_valid(params_json)),
  secret_ref TEXT,
  updated_at TEXT NOT NULL
);

CREATE TABLE model_sources (
  model_id TEXT NOT NULL REFERENCES models(id),
  source_id TEXT NOT NULL REFERENCES sources(id),
  source_key TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN ('active', 'suspect_missing', 'unavailable_confirmed')),
  first_seen_at TEXT NOT NULL,
  last_seen_at TEXT NOT NULL,
  miss_count INTEGER NOT NULL DEFAULT 0 CHECK(miss_count >= 0),
  context_length INTEGER, -- 来源报告的模型上下文能力，不是 offers 的价格档位范围
  updated_at TEXT NOT NULL,
  PRIMARY KEY(model_id, source_id),
  UNIQUE(source_id, source_key)
);

CREATE TABLE crawl_runs (
  id TEXT PRIMARY KEY NOT NULL,
  trigger TEXT NOT NULL CHECK(trigger IN ('schedule', 'manual', 'replay')),
  status TEXT NOT NULL CHECK(status IN ('running', 'success', 'partial', 'failed', 'abandoned')),
  started_at TEXT NOT NULL,
  heartbeat_at TEXT NOT NULL,
  finished_at TEXT,
  error TEXT
);

CREATE TABLE source_runs (
  run_id TEXT NOT NULL REFERENCES crawl_runs(id),
  source_id TEXT NOT NULL REFERENCES sources(id),
  status TEXT NOT NULL CHECK(status IN ('success', 'empty', 'incomplete', 'error')),
  is_complete INTEGER NOT NULL CHECK(is_complete IN (0, 1)),
  items_found INTEGER NOT NULL CHECK(items_found >= 0),
  attempts INTEGER NOT NULL CHECK(attempts > 0),
  raw_manifest TEXT NOT NULL,
  finished_at TEXT NOT NULL,
  error TEXT,
  PRIMARY KEY(run_id, source_id)
);

CREATE TABLE sync_outbox (
  id TEXT PRIMARY KEY NOT NULL,
  entity TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  payload_json TEXT NOT NULL CHECK(json_valid(payload_json)),
  created_at TEXT NOT NULL,
  attempts INTEGER NOT NULL DEFAULT 0,
  next_retry_at TEXT,
  error TEXT
);
```

1. 网络抓取、分页和校验在事务外完成，raw 先完整落盘，不在网络等待期间持有写事务。
2. 一个本地事务内应用已通过校验的来源数据、记录价格变化，并写入待同步队列；事务失败时一起回滚，避免本地数据已变而同步队列遗漏。
3. 同一个来源的多页结果先合并后保存。价格未变只更新最后观测时间；某来源失败时保留旧价格，只更新运行状态和 stale 标记，并将状态变化排入同步队列；其他成功来源照常入库。
4. 上传成功后删除对应队列项；失败时保留原 payload 和 ID，下一次自动重试。
5. 全部启用来源成功为 success，部分成功为 partial，全部失败为 failed；运行结果保存在 `crawl_runs` 和 `source_runs`。

## 六、采集结果完整性

这一节的目的很简单：避免抓取失败被误认为模型已经下架。

- 每个来源说明自己能否覆盖完整模型/报价列表。只有请求、分页和字段校验都成功，才算一次完整抓取。
- 来源失败或只抓到部分页面时，保留旧数据并标记 stale；不能因本次没看到某个模型就把它删除。
- 完整列表连续三次都没有某模型时，标记为“可能已下架”；第一版不自动永久删除，需管理员确认。
- 不同来源和不同价格档位分别保存。默认展示优先级最高且数据最新的有效来源，历史按具体报价查看。

## 七、币种换算

这一节只服务于 USD/CNY 展示，不参与来源价格判断。

- 原始报价始终按来源币种保存；每日抓取时更新 USD/CNY 汇率，并在页面显示汇率日期。
- 汇率变化不生成厂商调价记录；历史记录保留抓取时使用的汇率，缺少汇率时仍展示原币价格。

## 八、同步到 Cloudflare D1

同步的作用是把 NAS SQLite 中的公开数据送到 D1，供 Cloudflare Worker 和网页读取。它不是备份。

1. 每次来源结果通过校验并写入 SQLite 时，同一事务把需要公开的变更放进 `sync_outbox`。
2. 常驻服务随后把队列记录按来源、模型、报价、价格历史的顺序写入 D1。使用参数化 SQL、稳定 ID 和幂等 upsert；确认 D1 接收成功后才移除队列记录，网络失败时保留并自动重试。
3. Cloudflare Worker 直接查询 D1 当前数据。同步过程中页面可能短暂读到部分更新；首版接受这一点，若实际需要整轮原子切换再单独设计。
4. D1 仅含公开数据。源站密钥、来源私有配置、运行日志和 raw 文件仍保存在 NAS。

## 九、公开 API 与前端

| 接口 | 行为 |
|------|------|
| GET /api/models?currency=USD\|CNY | 从 D1 读取当前报价，可选币种 |
| GET /api/models/:id/history?offer_id=...&currency=USD\|CNY | 读取指定报价历史，可附日期范围、limit、cursor |
| GET /healthz | 返回 Worker 存活状态 |

响应示例，金额使用十进制整数字符串：

```json
{
  "amount_scale": 1000000,
  "unit": "per_1m_tokens",
  "models": [{
    "id": "0123456789abcdef0123456789abcdef",
    "provider": "openai",
    "display_name": "Example Model",
    "offers": [{
      "id": "1123456789abcdef0123456789abcdef",
      "source_id": "2123456789abcdef0123456789abcdef",
      "mode": "standard",
      "context_min": 0,
      "context_max": 0,
      "currency": "USD",
      "status": "active",
      "input_price": "2500000",
      "output_price": "10000000",
      "cached_input_price": null,
      "cache_write_price": null,
      "last_seen_at": "2026-11-17T00:59:00.000Z",
      "fx_rate": "7.10",
      "fx_as_of": "2026-11-17"
    }]
  }]
}
```

- 非法 ID、币种和范围返回 400；模型或报价不存在返回 404；D1 不可用返回 503。
- Worker 只公开 `/api/*` 和静态资源；来源配置入口留在 NAS，不暴露到公开站点。

## 十、Docker、CLI 与调度

Docker 镜像作为常驻服务运行，Python 调度器每天北京时间 09:00 触发抓取；每轮完成后尝试同步 D1，并按日生成 SQLite 备份。服务异常退出后由 Docker 重启；NAS 主机不另设 Cron。手动抓取与定时抓取调用同一套流程。

首版 Python 3.13 slim 系列，构建时固定版本/digest；发布前验证实际 NAS 架构。GitHub Actions 构建 linux/amd64 和 linux/arm64 镜像并合并为同一 GHCR 地址，NAS 不需要 Python 开发环境或自行构建镜像。

| CLI 命令 | 行为 |
|----------|------|
| serve | 启动常驻服务和内置调度器 |
| run-once | 立即抓取一次，之后同步已入库数据 |
| sync | 重试尚未同步到 D1 的公开数据 |
| sources import-defaults / import / export | 首次导入镜像内置种子；导入校验合并；导出不含凭据 |
| sources confirm-unavailable | 管理员确认某模型已停止供应 |
| status | 查看服务状态、最近抓取时间和待同步数量 |

- 服务只运行一个实例；抓取、手动操作和未来的配置界面共用同一数据库，并通过进程锁避免同时写入。
- 收到 SIGTERM 时停止接收新任务，等待当前来源在宽限时间内结束；Docker 使用 `restart: unless-stopped`。
- 服务保存最近一次运行状态和待同步数量，失败的同步在后续调度周期自动重试。
- HTTP 默认连接 10 秒、读取 30 秒，另设源总时限。最多三次抓取，同域串行且间隔至少一秒，遵守 Retry-After。重定向最多五次，解压后单响应最多 10 MiB；截断/超限/失败页不得作为定价页。
- 配置导入按稳定 source_id 合并，未列出的源保留，停用显式 disable。名称冲突、未知 parser、非法 URL/参数使整次导入失败。参数按 parser 自己的 schema 校验，不承诺任意 JSON API 无代码接入。
- 后续的 `apps/admin-web/` 运行在 NAS 局域网，通过本地 API 修改 SQLite 中的来源和调度配置；首版调度时间固定为每天 09:00，届时可由界面修改。它与 Cloudflare 上的公开 `apps/web/` 分开部署。首版不映射 Web 端口，密钥仍通过 NAS 运行环境管理。

Compose 设计示例（实施时写入仓库根目录 `compose.yaml`）：

NAS 的 .env 中设置 `LLM_RADAR_IMAGE=ghcr.io/lblbk/llm-radar:v0.1.0`（示例版本，须已发布），或使用 Release 输出的完整 digest 引用；其余数据路径和 UID/GID 仍由 NAS 本地填写。

```yaml
services:
  llm-radar:
    image: ${LLM_RADAR_IMAGE:?set a versioned image}
    restart: unless-stopped
    user: "${NAS_UID:?set uid}:${NAS_GID:?set gid}"
    init: true
    read_only: true
    environment:
      TZ: Asia/Shanghai
      LLM_RADAR_DATA_DIR: /data
      LLM_RADAR_BACKUP_DIR: /backup
    env_file:
      - .env
    volumes:
      - ${NAS_DATA_DIR:?set absolute local path}:/data
      - ${NAS_BACKUP_DIR:?set NAS backup path}:/backup
    tmpfs:
      - /tmp
    stop_grace_period: 60s
    command: ["serve"]
```

- 镜像 ENTRYPOINT 是已安装 CLI，依赖按锁文件 frozen 安装；.dockerignore 排除数据、raw、密钥和开发环境。
- /data 挂 NAS 本机文件系统专用目录并赋给指定 UID/GID，SQLite 主库不放 SMB/NFS。参见 [Docker bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)。
- `/backup` 映射到单独配置的 NAS 备份位置；服务每天将 SQLite 一致性副本写入该目录。
- `.env` 仅在运行时加载，不写入镜像。源站密钥和 D1 写入令牌不进入日志。

## 十一、原始数据、备份与恢复

```text
/data/
├── crawl.db
├── raw/<run_id>/<source_id>/<attempt>/<page_id>.body.gz
├── raw/<run_id>/<source_id>/manifest.json
└── logs/YYYY-MM-DD.jsonl
```

- D1 是网站使用的公开数据副本，不含 NAS 的来源私有配置、运行记录和待同步队列，因此不能代替本地备份。
- 每天用 SQLite backup API 将 `crawl.db` 备份到 NAS 上单独配置的备份位置，保留最近 7 份。备份内容包含来源配置和待同步队列；`.env` 中的密钥另行安全备份。
- raw 只用于排查和重放解析，默认保留最近 7 天，不要求放进数据库备份；无法恢复时可重新抓取来源。
- 备份位置应使用不同存储卷；NAS 已有可靠快照备份时，也可使用其备份策略。首次升级前验证能从一份备份恢复 SQLite。

## 十二、鉴权、额度与监控

- D1 写入令牌只保存在 NAS 的运行环境中，只授予所需数据库权限；公开 Worker 不提供写入或配置接口。
- 常驻服务不映射公网端口。后续配置界面只在 NAS 局域网开放，并使用本地 API 修改配置。
- 监控服务是否运行、最近一次抓取结果、来源连续失败、待同步队列长度、D1 API 错误、磁盘空间和最近一次备份时间。
- 上线前核对 Cloudflare 当前 D1 和 Workers 使用限制；缓存是否需要留到有实际读取数据后再决定。

## 十三、测试与实施顺序

1. 完成 SQLite/D1 公共表和来源配置结构。
2. 用 1～2 个真实来源跑通抓取，fixture 覆盖分页、空响应、缺字段和价格单位。
3. 验证常驻服务、定时触发、容器重启和手动抓取不会重复写入。
4. 验证 D1 同步可重试、重复上传不会产生重复历史记录，Worker 能读取当前数据。
5. 验证 SQLite 备份可恢复，再完成 NAS 实际部署和镜像升级流程。
6. 后续阶段增加 NAS 局域网配置界面；公开 Cloudflare 页面继续只读。

部署时填写 NAS 架构/UID/GID、数据路径、备份位置、Cloudflare D1 信息、D1 写入令牌和告警地址。

## 十四、GitHub Actions 与 GHCR 分发

### 14.1 固定镜像入口

根据当前 Git remote，源码仓库为 [lblbk/llm-radar](https://github.com/lblbk/llm-radar)，计划镜像命名如下；镜像在第一次成功发布后才可拉取。

| 用途 | 地址 / 引用 |
|------|-------------|
| 镜像入口 | `ghcr.io/lblbk/llm-radar` |
| 指定版本示例 | `ghcr.io/lblbk/llm-radar:v0.1.0` |
| 精确构建 | `ghcr.io/lblbk/llm-radar@sha256:<发布输出的索引 digest>` |
| GHCR 包页面 | https://github.com/lblbk/llm-radar/pkgs/container/llm-radar |

- 发布正式版本时统一使用 vMAJOR.MINOR.PATCH 标签。版本标签按项目策略禁止覆盖，重发发现已存在时核对/报错，不能悄悄替换；需要修改就发布新版本。digest 才是内容寻址的精确引用。
- 第一版仅分发明确版本，不自动维护 latest；README 和 NAS 配置都使用固定版本或 digest，升级是显式操作。
- 同一镜像索引包含 linux/amd64 和 linux/arm64，Docker 按 NAS 平台选择；digest 使用多架构索引的 digest，不能把 amd64 子镜像 digest 当成通用地址。参见 [Docker 多平台 Actions](https://docs.docker.com/build/ci/github-actions/multi-platform/)。

### 14.2 CI 与发布流程

1. PR 和默认分支 push 运行 ci.yml，验证 Python、共享契约、迁移及 Worker/前端构建。PR 只测试或试构建，不登录 GHCR、不推送镜像。
2. publish-image.yml 以 `release: types: [published]` 触发；首版只接受非 draft、非 prerelease 的正式 vMAJOR.MINOR.PATCH 版本。workflow 明确检出 Release 对应 tag 的源码提交，在该提交重新运行发布所需测试，不拿默认分支状态代替。
3. 测试通过后使用 Buildx 和 QEMU（或双架构原生 runner）构建，context 为仓库根目录、Dockerfile 为 `apps/llm-radar/Dockerfile`，让镜像能包含 `apps/llm-radar/config/`、`apps/shared/database/migrations/` 和 `apps/shared/contracts/`。根 `.dockerignore` 排除 .git、raw、本地库、.env、备份和不需要的前端开发产物。
4. 两种架构均完成构建及常驻服务启动检查后，推送同一个版本的多架构索引到 GHCR。任一架构失败不宣布版本发布完成；不能只构建 amd64 就声称支持 ARM NAS。
5. 发布 job 使用 GITHUB_TOKEN 登录 ghcr.io，权限仅 contents:read、packages:write；若工作流自动编辑 Release 说明，再仅给该 job contents:write。D1 写入令牌和源站凭据不进入构建流程。参见 [GitHub 镜像发布](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)。
6. OCI 标签记录 source=https://github.com/lblbk/llm-radar、revision=实际源码提交、version=Release tag，将包关联到仓库。第三方 Actions 固定审核过的完整 commit SHA，版本更新时显式维护。
7. 成功 job summary 输出镜像 tag、源码 commit、架构、索引 digest、拉取命令；Release 提供相同结果或该次成功运行链接。GitHub Release 事件本身早于构建成功，README 不能把已创建 Release 等同于镜像已可用。
8. CF Worker/公开网页部署保持独立 workflow；发布 NAS 镜像只更新常驻服务，不触发抓取或修改 NAS 数据。

### 14.3 可见性与 NAS 升级

- 目标是公共镜像，供 NAS 使用一个 GHCR 引用直接拉取。GHCR 包初次发布默认 private，仓库公开不代表包自动公开；首次发布后在包设置显式改为 Public，并验证匿名拉取。参见 [GitHub Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)。
- 如果以后改为私有包，NAS 另外配置具有 read:packages 权限的 PAT classic 登录；该凭据不是 Actions 的 GITHUB_TOKEN，也不是 D1/Access token。
- 首次部署下载对应版本的根目录 `compose.yaml` 和 `.env.example`，填写 NAS 数据目录、备份目录、凭据与镜像版本，拉取镜像后导入默认来源并启动常驻服务。无需在 NAS clone 整个仓库或安装构建工具。
- 升级前确认 SQLite 备份可用；更新 `LLM_RADAR_IMAGE` 后拉取并重建服务。Docker 会先优雅停止旧容器，再启动新版本。

```sh
docker compose pull llm-radar
docker compose up -d llm-radar
docker compose logs --tail 100 llm-radar
```

- 容器重建必须保留 `/data` 和 `/backup` 挂载；换镜像不能重新初始化或删除 SQLite 数据。
- 回退时恢复旧镜像版本；只有数据库结构不兼容时才从升级前备份恢复。
