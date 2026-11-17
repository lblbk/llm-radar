# llm-radar 实施方案 v2

NAS 使用 Python 抓取，以 Docker 单次任务运行；本地 SQLite 是业务事实来源。源码托管 GitHub，GitHub Actions 构建并发布 GHCR 镜像，NAS 拉取固定版本即可运行。Cloudflare 保留 TypeScript Worker、D1、KV 和 React 展示站点，NAS 无须暴露公网端口。

## 一、架构与职责

```text
GitHub：lblbk/llm-radar → Actions 测试/构建 → ghcr.io/lblbk/llm-radar
                                            ↓ NAS 拉取固定版本
NAS 任务计划（每天一次，Asia/Shanghai）
  → Docker：Python CLI，持有 /data/task.lock
  → Fetcher → 原始响应落盘 → Parser → 完整性校验 → SQLite
  → 同事务保存业务变更、完整目录快照、sync_outbox
  → Syncer 按批次顺序上传 D1 HTTP API
  → 全部分块确认完成 → 写 D1 releases 完成标记
  → POST /admin/snapshot → Worker 发布 KV latest

浏览器 → Workers Assets：React SPA
       → GET /api/models：KV 中某一完整版本
       → GET /api/models/:id/history?version=N：截至 N 的 D1 历史

NAS 主库 + raw + 待同步记录 → 一致性备份 → 第二位置
```

| 部署侧 | 职责 |
|--------|------|
| GitHub / Actions / GHCR | 源码、测试、多架构镜像构建与分发，不运行生产抓取任务 |
| NAS / Python | 配置、抓取、解析、归一化、价格事件、源状态、汇率、备份、上传和发布重试 |
| Worker / TypeScript | 公开查询、校验并发布快照、静态资源路由 |
| D1 | 公开数据副本、不可变价格事件、完整版本目录 |
| KV | 一个完整的 latest 响应对象，允许最终一致的传播延迟 |
| React | 模型列表、来源选择、币种切换、历史阶梯曲线 |

- CF 不抓取、不调度业务任务、不编辑源配置，不保存源站认证信息。
- Worker 唯一管理入口是 `/admin/snapshot`，只写 KV；业务上传走 D1 HTTP API。
- 上传、快照发布、网页读取分别记录状态；抓取成功不等于发布成功。
- Git 不存运行数据、密钥、本地数据库或真实响应；测试只提交必要的脱敏样本。

## 二、技术栈与目录

Python 使用 httpx、Pydantic、标准库 sqlite3 / decimal、HTML 解析库和 pytest。优先 JSON/API，再解析 HTML；确实依赖 JavaScript 的源使用可选浏览器镜像。第一版串行抓取，不引入 FastAPI、Redis、Celery 或容器内第二套调度器。

```text
llm-radar/
├── README.md
├── .github/workflows/
│   ├── ci.yml                     # PR / 默认分支测试，不发布 PR 镜像
│   └── publish-image.yml          # Release 发布 GHCR 镜像
├── .dockerignore                  # 根目录是 Docker build context
├── wrangler.jsonc                 # D1 / KV / Assets，无抓取 Cron
├── migrations/
│   ├── shared/                    # SQLite、D1 公共表迁移
│   └── nas/                       # 仅 SQLite：配置、运行、outbox
├── contracts/
│   ├── item.schema.json
│   ├── parser-result.schema.json  # 完整性、分页、校验结果
│   ├── catalog.schema.json
│   ├── api.schema.json
│   └── examples/                  # Python / TS 共用正反例
├── default-sources.json           # 首次 CLI 导入，不自动覆盖配置
├── nas/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── Dockerfile
│   ├── compose.yaml
│   ├── .env.example               # 无敏感信息
│   ├── src/llm_radar/
│   │   ├── cli.py
│   │   ├── config.py
│   │   ├── fetcher.py
│   │   ├── parsers/
│   │   ├── normalize.py
│   │   ├── store.py
│   │   ├── fx.py
│   │   ├── syncer.py
│   │   └── backup.py
│   └── tests/fixtures/
├── src/
│   ├── index.ts                   # Hono
│   ├── api/routes.ts
│   ├── snapshot.ts
│   └── repo/readonly.ts
└── web/                           # Vite / React / TypeScript
```

- 取消跨语言的 shared/fetcher.ts 和 TS parser；Worker 不需要抓取实现。
- 跨语言共享 JSON Schema、示例和公共迁移；Python/TS 分别校验，CI 验证同一批样本。
- migrations 是表结构事实来源，不再同时手写 schema.sql。SQLite 和 D1 分别记录迁移版本。
- 公共迁移先验证 SQLite 和 D1 开发库；上线先做向后兼容的远端扩展，再升级 NAS/Worker，最后移除旧字段。迁移失败禁止同步，不自动删库重建。

## 三、身份、金额与定价维度

1. 源、模型、报价、价格事件、运行和批次 ID 均由 NAS 生成 UUID 的 32 位小写十六进制字符串，两边原样保存。D1 不自行分配实体 ID。
2. 模型公开 URL 使用 models.id，不把含斜杠的厂商名称当路径参数。原始名称保留大小写；canonical_name 由厂商适配器确定，不能统一 lowercase 后猜测别名。
3. provider 使用固定厂商标识，展示名另存。跨源别名通过显式映射归并，没有把握的不自动合并。
4. 一个报价序列由模型、来源、模式（standard / batch）、上下文下限/上限、原币种唯一确定。上下文采用 `[min, max)`，max=0 表示无上限；重叠或不支持的档位拒绝入库。
5. 输入、输出、缓存读取、缓存写入分别存金额；缺失为 NULL，免费为 0。音频、图片、按请求收费等不强行换算 token，第一版标为不支持并告警。
6. 金额统一为“每百万 token 的原币金额 × 1,000,000”的整数。例如每百万 token 2.5 USD 存 2500000。Python 用 Decimal 从原始十进制字符串转换，禁止先转 float；超出精度或整数范围时拒绝静默舍入。
7. JSON 中金额用十进制整数字符串，避免 JS 整数精度丢失。格式化、换算和显示舍入不参与调价判定。
8. 业务时间由 NAS 生成 UTC ISO 8601 毫秒值，远端原样保存；NAS 调度按北京时间，汇率日期按 UTC。

## 四、公共数据模型

以下是首个公共 migration 的设计基线。SQLite 每个连接启用 `PRAGMA foreign_keys=ON`。batch_seq 为 NAS 持久化分配的递增发布批次编号，不是实体 ID。

```sql
CREATE TABLE sources (
  id TEXT PRIMARY KEY NOT NULL,
  name TEXT NOT NULL UNIQUE,
  public_url TEXT,
  priority INTEGER NOT NULL DEFAULT 100,
  enabled INTEGER NOT NULL CHECK(enabled IN (0, 1)),
  revision_seq INTEGER NOT NULL CHECK(revision_seq > 0),
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE models (
  id TEXT PRIMARY KEY NOT NULL,
  provider TEXT NOT NULL,
  canonical_name TEXT NOT NULL,
  display_name TEXT NOT NULL,
  revision_seq INTEGER NOT NULL CHECK(revision_seq > 0),
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

-- 不可变事件；每批每个报价最多一个最终事件。
CREATE TABLE price_history (
  id TEXT PRIMARY KEY NOT NULL,
  offer_id TEXT NOT NULL REFERENCES offers(id),
  batch_seq INTEGER NOT NULL CHECK(batch_seq > 0),
  input_price INTEGER CHECK(input_price >= 0),
  output_price INTEGER CHECK(output_price >= 0),
  cached_input_price INTEGER CHECK(cached_input_price >= 0),
  cache_write_price INTEGER CHECK(cache_write_price >= 0),
  fx_rate_id TEXT REFERENCES fx_rates(id),
  captured_at TEXT NOT NULL,
  CHECK(input_price IS NOT NULL OR output_price IS NOT NULL
     OR cached_input_price IS NOT NULL OR cache_write_price IS NOT NULL),
  UNIQUE(offer_id, batch_seq)
);
CREATE INDEX idx_price_history
  ON price_history(offer_id, captured_at, batch_seq, id);

-- 整批确认完成后最后写入。价格行允许先上传，但未完成版本不可查询。
CREATE TABLE releases (
  seq INTEGER PRIMARY KEY CHECK(seq > 0),
  id TEXT NOT NULL UNIQUE,
  parent_seq INTEGER REFERENCES releases(seq),
  schema_version INTEGER NOT NULL,
  generated_at TEXT NOT NULL,
  catalog_json TEXT NOT NULL CHECK(json_valid(catalog_json)),
  catalog_sha256 TEXT NOT NULL,
  manifest_json TEXT NOT NULL CHECK(json_valid(manifest_json))
);
```

- sources 只含公开元信息。抓取 URL、headers、参数、secret 引用放 NAS 专用配置表；认证参数、错误堆栈、原始响应不上传。
- offers 定价维度和价格事件不可变；换币种/档位创建新序列。最新可用报价、失效档位由完整目录明确给出，不能取任意历史行冒充当前价。
- 模型和源元信息可以更新，但公开接口从所选版本目录读取，不能直接读取已上传到下一版本的业务表状态。
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
  context_length INTEGER,
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

CREATE TABLE offer_state (
  offer_id TEXT PRIMARY KEY NOT NULL REFERENCES offers(id),
  status TEXT NOT NULL CHECK(status IN ('active', 'suspect_missing', 'unavailable_confirmed')),
  last_seen_at TEXT NOT NULL,
  last_event_id TEXT NOT NULL REFERENCES price_history(id),
  updated_at TEXT NOT NULL
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

CREATE TABLE sync_batches (
  seq INTEGER PRIMARY KEY AUTOINCREMENT,
  id TEXT NOT NULL UNIQUE,
  run_id TEXT REFERENCES crawl_runs(id),
  status TEXT NOT NULL CHECK(status IN ('pending', 'uploading', 'uploaded', 'published', 'blocked')),
  snapshot_pending INTEGER NOT NULL DEFAULT 1 CHECK(snapshot_pending IN (0, 1)),
  created_at TEXT NOT NULL,
  uploaded_at TEXT,
  published_at TEXT,
  attempts INTEGER NOT NULL DEFAULT 0,
  next_retry_at TEXT,
  error TEXT
);

CREATE TABLE sync_outbox (
  batch_seq INTEGER NOT NULL REFERENCES sync_batches(seq),
  ordinal INTEGER NOT NULL,
  entity TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  payload_json TEXT NOT NULL CHECK(json_valid(payload_json)),
  payload_sha256 TEXT NOT NULL,
  acknowledged INTEGER NOT NULL DEFAULT 0 CHECK(acknowledged IN (0, 1)),
  PRIMARY KEY(batch_seq, ordinal),
  UNIQUE(batch_seq, entity, entity_id)
);
```

1. 网络抓取、分页和校验在事务外完成，raw 先完整落盘，不在网络等待期间持有写事务。
2. 一个本地事务内创建批次、应用合格源数据、计算价格事件和源状态、生成完整目录、保存全部 outbox 与本地 release。异常全部回滚，避免业务已提交但 outbox 丢失。
3. 同实体在批次内合并为最终状态；每报价本轮最多一个事件。价格未变只更新 model_sources / offer_state 的观测信息；一个源失败不影响其他合格源，失败源保留旧数据并标 stale。新目录在上一份本地目录及持久状态基础上更新，不能依赖进程内记忆恢复档位。
4. 批次封存后 payload、事件 ID、业务时间和 hash 不再变化；重试读取 outbox，不能从当前业务表重新拼旧批次。
5. sync_batches 保存传输/发布进度，不用 last_sync_at 推测遗漏。配置修改、汇率更新同样生成批次。
6. 全部启用源合格为 success，部分合格为 partial，全部失败为 failed；没有启用源是成功空操作，不制造报价。所有失败仍记录运行结果和旧目录健康度。

## 六、完整性、下架与来源选择

- Parser 是纯函数 `parse(raw_pages, params) -> ParseResult`，不联网、不访问 DB。结果包含条目、分页完整性、覆盖范围、错误和警告；结合 fetcher 分页清单验证。
- 单源整份结果通过校验才写业务数据。重复标识、缺页、截断、单位不支持、关键字段缺失均记 incomplete/error，禁止部分写入。
- 空结果记 empty，最多三次抓取后仍为空则告警，不更新缺失计数。异常大幅条目下降也进入核查，不能仅凭 items_found 大于零确认目录完整。
- 只有声明覆盖完整模型目录、所有页成功且校验通过的源，才对未出现的 model_sources 增加 miss_count。部分覆盖源永不驱动全局下架。
- 连续三次合格目录抓取未见，关联标为 suspect_missing；第一版不据此自动永久退休模型。
- 源站明确停止供应，或管理员 CLI 确认，才能将关联设 unavailable_confirmed。至少一个启用源仍 active 时全局模型 active；所有启用关联均 confirmed 才 retired；只有疑似或没有启用关联时 unknown。停用源不代表模型下架。
- 再次看到模型自动恢复关联 active 并清零缺失；全局状态随关联重新派生。禁用源不累计缺失。
- 报价档位也跟踪可用性：完整报价清单不再出现的 offer 从当前目录移除，历史保留；部分覆盖响应不能确认档位停止供应。
- 默认来源按 priority、健康度、稳定 ID 排序，优先新鲜 active 官方源；允许查看其他来源。历史选择具体 offer，不把不同来源/档位连成同一条曲线。

## 七、汇率和显示

- 默认 USD，可切 CNY。usd_cny 是精确十进制字符串，固定语义 `1 USD = rate CNY`，必须为正有限值。
- 抓取开始时尝试更新日汇率；配置指定 provider、endpoint、响应映射并保留有效报价日期，不把请求日期冒充节假日实际报价日期。
- 汇率服务独立于价格入库；移除原先的 fx_rate_to_usd / fx_rate_at。汇率只存 fx_rates，事件用 fx_rate_id 关联，不重复存汇率金额快照。
- 列表使用目录生成时最近可用汇率。允许最多 7 个日历日的旧值，标注日期和 estimated；无合格汇率则跨币种金额 null、fx_status=unavailable，仍返回原币报价，不显示为零。
- 新价格事件固定关联 `as_of <= UTC date(captured_at)` 的最近合格汇率，没有则 NULL。首次接受的日汇率、事件关联均不可变，保证历史可复现。
- 历史按事件固定汇率，USD→CNY 乘 rate，CNY→USD 除 rate；原币展示无需汇率。后补汇率不悄悄改写缺汇率的旧事件。
- 列表按最新汇率、历史按事件汇率，末端可能不同，UI 分别标注汇率日期和折算含义，不将汇率变化描述成厂商调价。
- 曲线按观测时间、批次、事件 ID 稳定排序，采用阶梯形式，可延伸最后一档到版本观测时间。无变化日期不制造调价事件，不把观测时间冒充官方生效时间。

## 八、同步与版本发布

### 8.1 HTTP API 与写入

```text
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/d1/database/{database_id}/query
Authorization: Bearer <D1_API_TOKEN>
Content-Type: application/json
```

```json
{
  "batch": [{
    "sql": "INSERT INTO models (id, provider, canonical_name, display_name, revision_seq, created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?, ?) ON CONFLICT(id) DO UPDATE SET display_name=excluded.display_name, revision_seq=excluded.revision_seq, updated_at=excluded.updated_at WHERE excluded.revision_seq > models.revision_seq",
    "params": ["0123456789abcdef0123456789abcdef", "openai", "example-model", "Example Model", 17, "2026-11-17T00:00:00.000Z", "2026-11-17T00:00:00.000Z"]
  }]
}
```

- /query 支持单条 sql/params 和参数化 batch；不另造 /batch 路径。表名来自程序白名单，值参数化，不拼配置到 SQL。
- 外键上传顺序：sources、models、offers、fx_rates、price_history，最后 releases。实体 ID 和 created_at/updated_at 原样传，不用远端 now 重写。
- 可变 sources / models 携带 revision_seq，只有更高版本能更新；同版本须核对内容相同，低版本不得覆盖。这样即使超时旧请求迟到，也不会把副本元信息改回旧值。provider/canonical_name 作为身份不原地改写，别名显示变更与身份迁移分开处理。
- 不可变记录重推时核对内容，一致才成功；同 ID 不同内容或自然键撞另一个 ID 必须 blocked。禁止无目标 INSERT OR IGNORE 掩盖错配。
- 检查 HTTP、顶层 success、每条 result.success 及错误数组；HTTP 200 不等于全部成功。超时后可能已提交，用相同 ID、参数、时间和 hash 重推并核验。
- 每分块初始最多 50 条，另满足单 SQL 字节、绑定参数、请求大小限制。跨请求没有事务，不能把分块成功当作整轮原子完成。单请求失败语义纳入 D1 集成测试。

### 8.2 持久化队列与重试

1. 持锁取最早未完成批次，先前批次未 published 时不上传后续批次。抓取可继续封存后续批次，但传输严格先进先出，禁止旧状态覆盖新状态。
2. 分块成功后持久化 acknowledged，重启后从 outbox 恢复。payload 中保存的是已冻结的实体值，不保存任意可执行 SQL。
3. 网络错误、429、可重试 5xx 使用随机抖动指数退避，遵守 Retry-After。单次任务最多尝试五次并受总时限约束；耗尽后保存 next_retry_at，下次 sync 继续。
4. 认证、契约、外键或内容冲突记 blocked，停止推进并告警；修复后重试原批次，不跳过已有事件。
5. 全部分块确认后，核对本批事件数量与预期，按稳定 ID 核验不可变内容；最后写 releases 完成标记，目录 hash 与 manifest 必须一致。
6. 远端 release 成功后本地改 uploaded，仍保持 snapshot_pending=1；发布接口确认版本和 hash 后才 published、snapshot_pending=0。发布失败持续重试，不重新抓取或重造批次。

### 8.3 可见版本与一致性

- 每批次保存完整公开目录到 releases.catalog_json，含元信息、来源状态、当前报价、汇率、stale 标记、最后事件 ID。目录不是增量。首版未压缩 UTF-8 上限 1 MiB，超限拒绝封存并告警；扩容另行引入分页目录协议。
- catalog_json 和 outbox payload 均由 NAS 按确定规则生成 UTF-8 JSON（键排序、无无关空白、禁止非有限数，精确金额/汇率用字符串），hash 针对保存的原始字节。Worker 验证原文 hash，不重新序列化后猜测相同 hash。manifest 记录父版本、各类实体/事件数量和冻结 payload 摘要。
- NAS POST /admin/snapshot body 为 version、catalog_sha256。Worker 只接受 D1 最新完整 release，检查前序链、契约、hash、体积；明确过期请求返回 409。NAS 持同一进程锁串行调用，此入口不允许其他并发发布者。
- Worker 读取冻结 catalog，不从正在变化的业务表现场重建。一次写入完整 KV latest，重复发布同版本同 hash 幂等，响应确认版本/hash。
- KV 各地区可能暂时读到旧版，但每次都是完整对象。不能拆成版本指针和目录两个 key 后假设原子更新，也不能承诺即时全球生效。
- KV 没有条件写入，本方案不承诺超时迟到请求下的严格单调版本。Worker 写入前再次检查最新 release；每次 sync 即使无待上传批次，也检查并重发最新完整快照以收敛迟到写入。前端已见新版本后可忽略较旧列表响应并重试。严格单调若成为需求，须增加串行发布协调器，不能仅靠 KV 的读后写校验声称已解决。
- 历史请求显式传 data_version，验证 release 存在且模型/offer 属于其目录，再查询该报价 batch_seq 不超过 version 的事件；事件批次必须属于已完成前序链。尚无完成 release 的半批数据不可查询。
- 模型名、档位、可用性从所选版本目录读取，汇率从事件固定关联读取；缓存键含 version、offer、币种、游标和范围。
- 无 KV 快照返回 503 snapshot_unavailable，不返回伪成功空列表，不回退到无版本的实时 D1 扫描。重建从最新完整 release 恢复，不凭本地发送时间猜版本。

## 九、API 与前端契约

| 接口 | 行为 |
|------|------|
| GET /api/models?currency=USD\|CNY | 默认 USD，读取 KV 完整目录并派生展示金额 |
| GET /api/models/:id/history?version=N&offer_id=...&currency=USD\|CNY | version、offer 必填，可附日期范围、limit、cursor |
| GET /healthz | 存活、快照版本和时间，不暴露内部错误 |
| POST /admin/snapshot | Access 机器身份保护，发布完整版本，不接受任意 catalog 上传 |

目录最小业务示例，金额是定点整数字符串：

```json
{
  "schema_version": 1,
  "data_version": 17,
  "generated_at": "2026-11-17T01:00:00.000Z",
  "amount_scale": 1000000,
  "unit": "per_1m_tokens",
  "fx": {"usd_cny": "7.10", "as_of": "2026-11-17", "estimated": false},
  "stale_sources": [],
  "models": [{
    "id": "0123456789abcdef0123456789abcdef",
    "provider": "openai",
    "display_name": "Example Model",
    "status": "active",
    "offers": [{
      "id": "1123456789abcdef0123456789abcdef",
      "source_id": "2123456789abcdef0123456789abcdef",
      "mode": "standard",
      "context_min": 0,
      "context_max": 0,
      "currency": "USD",
      "input_price": "2500000",
      "output_price": "10000000",
      "cached_input_price": null,
      "cache_write_price": null,
      "last_seen_at": "2026-11-17T00:59:00.000Z"
    }]
  }]
}
```

- 最终 JSON Schema 还须列明优先来源、last_event_id、源健康度、列表换算字段和缺汇率分支；目录与 API 响应分别校验，示例不等于全部字段。
- 非法 ID/币种/游标/范围返回 400，版本/模型/报价不存在返回 404，存储或契约不可用返回 503。历史 limit 默认 200、最大 1000，不提供无界查询。
- retired 默认隐藏但详情/历史可查；unknown、suspect_missing 保留提示。前端用列表版本查询历史，刷新版本时取消旧请求。
- 同币种显示原值，跨币种标汇率日期；missing 不变成零，stale 不被普通成功状态掩盖。
- /api/* 和 /admin/* 明确由 Worker 处理，其余走 Assets/SPA；API 404 不返回 index.html。设置 ETag，缓存不掩盖数据时间。

## 十、Docker、CLI 与调度

首版 Python 3.13 slim 系列，构建时固定版本/digest；发布前验证实际 NAS 架构。GitHub Actions 构建 linux/amd64 和 linux/arm64 镜像并合并为同一 GHCR 地址，原生依赖分别验证。NAS 不需要 Python 开发环境或自行构建镜像。浏览器抓取使用独立可选镜像。

| CLI 命令 | 行为 |
|----------|------|
| run | 抓取、封存，再按顺序同步/发布，最后备份 |
| crawl | 仅抓取、落库和封存，支持断网 |
| sync | 恢复上传和发布，不重新抓源 |
| sources import/export | 校验合并/导出，导出不含凭据 |
| sources confirm-unavailable | 确认某模型在某源停止供应，生成状态批次 |
| replay | 默认只预览解析差异，不改历史或缺失计数；确认当前观测需重新抓取 |
| backup / verify-backup | 一致性备份 / 隔离恢复验证 |

- 所有写库命令共用 /data/task.lock 进程锁，包括导入、sync、确认、备份轮转。Linux flock 由文件描述符持有，进程退出自动释放；不能用锁文件是否存在判断锁。
- NAS 每天北京时间 09:00 执行 `docker compose run --rm crawler run`，一次运行后退出，restart 为 no。可以加纯 sync 补传计划，但仍共用锁；容器内不再开 Cron。
- 总时限默认 30 分钟，单源 5 分钟，支持 SIGTERM、回滚未提交事务。下次拿到锁后将遗留 running 记 abandoned，已封存批次继续恢复。
- 退出码：0 目标成功或无待办；1 抓取部分/全部失败；2 配置/认证/存储或 blocked；3 批次尚未上传/发布；75 任务已在运行。多种失败优先 2、3、1。crawl-only 不因有正常待同步批次报 3。
- HTTP 默认连接 10 秒、读取 30 秒，另设源总时限。最多三次抓取，同域串行且间隔至少一秒，遵守 Retry-After。重定向最多五次，解压后单响应最多 10 MiB；截断/超限/失败页不得作为定价页。
- 配置导入按稳定 source_id 合并，未列出的源保留，停用显式 disable。名称冲突、未知 parser、非法 URL/参数使整次导入失败。参数按 parser 自己的 schema 校验，不承诺任意 JSON API 无代码接入。

Compose 设计示例（实施时写入 nas/compose.yaml）：

NAS 的 .env 中设置 `LLM_RADAR_IMAGE=ghcr.io/lblbk/llm-radar:v0.1.0`（示例版本，须已发布），或使用 Release 输出的完整 digest 引用；其余数据路径和 UID/GID 仍由 NAS 本地填写。

```yaml
services:
  crawler:
    image: ${LLM_RADAR_IMAGE:?set a versioned image}
    restart: "no"
    user: "${NAS_UID:?set uid}:${NAS_GID:?set gid}"
    init: true
    read_only: true
    environment:
      TZ: Asia/Shanghai
      LLM_RADAR_DATA_DIR: /data
    env_file:
      - .env
    volumes:
      - ${NAS_DATA_DIR:?set absolute local path}:/data
    tmpfs:
      - /tmp
    stop_grace_period: 60s
    command: ["run"]
```

- 镜像 ENTRYPOINT 是已安装 CLI，依赖按锁文件 frozen 安装；.dockerignore 排除数据、raw、密钥和开发环境。
- /data 挂 NAS 本机文件系统专用目录并赋给指定 UID/GID，SQLite 主库不放 SMB/NFS；所有容器必须挂同一宿主目录才能共享锁。参见 [Docker bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)。
- .env 权限 600，只运行时加载，不写镜像。源站 secret、D1 token、Access token 分开命名；日志不输出认证 headers。

## 十一、原始数据、备份与恢复

```text
/data/
├── task.lock
├── crawl.db
├── raw/<run_id>/<source_id>/<attempt>/<page_id>.body.gz
├── raw/<run_id>/<source_id>/manifest.json
├── logs/YYYY-MM-DD.jsonl
└── backup/<UTC-timestamp>/crawl.db + raw-manifest.json + checksums.json
```

- manifest 记录公开 URL、响应类型/编码/状态、UTC 抓取时间、hash、分页顺序、抓取器/解析器版本和完整性。不同运行/重试不覆盖，完整写临时文件后原子重命名。
- raw 可能含敏感内容，仅留在受控 NAS/备份位置，错误日志记脱敏摘要。
- 主库不是备份。默认 WAL、foreign_keys=ON、busy_timeout，备份使用 SQLite backup API；禁止写入中只复制 crawl.db 丢掉 WAL。参见 [Python sqlite3 backup](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.backup)。
- 备份含所有本地表/outbox，并关联 raw manifest；将数据库快照、manifest 及其引用的原始响应文件一并复制到第二块盘/外接盘/对象存储并验 hash，不能只备份文件清单。第二位置失败须告警，同盘 backup 不算已完成第二位置备份。
- 默认 raw 90 天、logs 30 天，日备份 30 份、周备份 12 份；未完成批次相关 raw、未备份/人工保留样本不按普通轮转删除。磁盘不足停止新抓取，不能清掉待补传记录。
- 每月临时目录恢复最近备份，做 integrity_check、foreign_key_check、outbox 数量/hash 和 raw 可读性检查，不覆盖生产。
- 恢复旧备份先核对 D1 最新 release 的 seq/ID/hash；远端领先时禁止直接续传或复用 seq，先找回更新备份/完整归档。无法补齐则新建数据集并标历史缺口，不声称旧 raw 一定能完整重建。
- D1 灾难恢复按 NAS 已发布批次重放；payload 和完整目录保留在数据库/备份中。清理已发布 outbox 前须另行实现可恢复归档协议。

## 十二、鉴权、额度与监控

- D1 API Token 只给所需账号 D1 写权限。发布接口另用 Cloudflare Access Service Token 和 Service Auth 策略，携带 CF-Access-Client-Id / CF-Access-Client-Secret，两类 token 不能互换。参见 [Access Service Tokens](https://developers.cloudflare.com/cloudflare-one/access-controls/service-credentials/service-tokens/)。
- 所有 Worker 可达域名都保护管理入口；关闭不用的 workers.dev/preview 入口，部署验收匿名调用必须被拒绝。公开响应只包含契约字段。
- NAS 只出站，不映射容器端口；401/403 立即告警，不无限重试。
- 等待数据库/KV/网络不算 Worker CPU；Worker 内解析、转换、校验、序列化消耗 CPU。保留 KV 是缓存和完整版本发布决策，不能声称任何 D1 读取都会耗尽 CPU。参见 [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/)。
- 预算覆盖普通 API 和快照 JSON/hash 校验，测试最大目录的 CPU/内存/延迟。Free 超限先缩减目录/分页再评估套餐，不保证永远免费。
- 同时监控 Workers 请求/CPU、D1 每日读写行数/存储、KV 操作量/值大小，索引和重试也有成本。上线复核 [D1 Limits](https://developers.cloudflare.com/d1/platform/limits/)、[D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/) 与 Workers/KV 官方限制。
- D1 HTTP 参数及结果以 [Query API](https://developers.cloudflare.com/api/resources/d1/subresources/database/methods/query/) 为准，不将 Worker 调用额度套给 NAS HTTP 上传。
- 结构化日志含 run_id、source_id、batch_seq、耗时和各阶段结果；告警覆盖任务未启动、源连续失败、条目异常下降、旧汇率、outbox 积压、版本未更新、磁盘不足、备份失败。
- 完整 run 的成功心跳仅在抓取、上传、发布、备份目标均达成后发送；partial/blocked 发失败信号。告警地址由 NAS 环境配置，未配置视为部署验收未完成。

## 十三、测试与实施顺序

1. 冻结 JSON Schema、正反例、版本规则和 migration；SQLite/D1 开发库分别验证，增加 UUID、金额、时间和参数的跨语言契约测试。
2. Python CLI 跑通 1～2 个真实源，fixture 覆盖币种/单位、分页、空响应、错页、缺字段、缓存价、档位。fixture 只验证已存样本，线上改版靠运行校验/告警发现。
3. 验证本地事务/outbox：异常整体回滚、进程终止恢复、同日重跑 raw 不覆盖、双容器仅一个写、重复解析不制造假调价。
4. D1 集成验证：ID 原样保存、同批重推、200 但子语句失败、超时已提交、外键/内容冲突、旧批次重试、冻结 payload 和进度恢复。
5. 发布验证：上传中读旧版、发布失败恢复、旧发布拒绝、KV 读旧版、历史版本过滤、无快照 503、目录大小/CPU 上限，不暴露半批目录或混合历史。
6. React 验证币种/来源/档位/版本、阶梯曲线、缺汇率、unknown/retired、缓存键、API 404/SPA 路由及匿名管理拒绝。
7. 实现 GitHub CI 与 GHCR 发布流程，测试后构建双架构镜像，验证公共包匿名拉取及 Release 中的地址/digest；实际 NAS 验证持久化、UID/GID、SIGTERM、退出码、计划任务、进程锁、断网补传和备份恢复，固定镜像版本后观察一周。

部署时填写：NAS 架构/UID/GID、数据绝对路径、第二备份位置、CF 账号/数据库/域名、两类机器凭据、实际汇率数据源、告警地址。调度、重试、轮转和配置管理按本方案默认值实施。

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

1. PR 和默认分支 push 运行 ci.yml，验证 Python、跨语言契约、迁移及相关前端/Worker 测试。PR 只测试或试构建，不登录 GHCR、不推送镜像。
2. publish-image.yml 以 `release: types: [published]` 触发；首版只接受非 draft、非 prerelease 的正式 vMAJOR.MINOR.PATCH 版本。workflow 明确检出 Release 对应 tag 的源码提交，在该提交重新运行发布所需测试，不拿默认分支状态代替。
3. 测试通过后使用 Buildx 和 QEMU（或双架构原生 runner）构建，context 为仓库根目录、Dockerfile 为 nas/Dockerfile，让镜像能包含公共 migrations 和 contracts。根 .dockerignore 排除 .git、raw、本地库、.env、备份和不需要的前端开发产物。
4. 两种架构均完成构建及 CLI/迁移冒烟检查后，推送同一个版本的多架构索引到 GHCR。任一架构失败不宣布版本发布完成；不能只构建 amd64 就声称支持 ARM NAS。
5. 发布 job 使用 GITHUB_TOKEN 登录 ghcr.io，权限仅 contents:read、packages:write；若工作流自动编辑 Release 说明，再仅给该 job contents:write。不需要为普通镜像推送另建长期 PAT，生产 D1/Access/源站凭据不进入构建流程。参见 [GitHub 镜像发布](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)。
6. OCI 标签记录 source=https://github.com/lblbk/llm-radar、revision=实际源码提交、version=Release tag，将包关联到仓库。第三方 Actions 固定审核过的完整 commit SHA，版本更新时显式维护。
7. 成功 job summary 输出镜像 tag、源码 commit、架构、索引 digest、拉取命令；Release 提供相同结果或该次成功运行链接。GitHub Release 事件本身早于构建成功，README 不能把已创建 Release 等同于镜像已可用。
8. 同一版本发布串行处理，不取消正在推送的发布任务；重试不得产生不同内容覆盖旧版本。CF Worker/网页部署保持独立 workflow，发布 NAS 镜像不执行抓取或更改 NAS 数据库。

### 14.3 可见性与 NAS 升级

- 目标是公共镜像，供 NAS 使用一个 GHCR 引用直接拉取。GHCR 包初次发布默认 private，仓库公开不代表包自动公开；首次发布后在包设置显式改为 Public，并验证匿名拉取。参见 [GitHub Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)。
- 如果以后改为私有包，NAS 另外配置具有 read:packages 权限的 PAT classic 登录；该凭据不是 Actions 的 GITHUB_TOKEN，也不是 D1/Access token。
- 首次部署下载对应版本的 nas/compose.yaml 和 .env.example，填写 NAS 路径/凭据与镜像版本，拉取后先验证 CLI，再导入种子配置。无需在 NAS clone 整个仓库或安装构建工具。
- 升级先暂停定时任务，确认任务结束并完成一致性备份，检查迁移兼容性；修改 LLM_RADAR_IMAGE 为新版本，依次执行下面命令，再恢复调度。NAS 必须能访问 ghcr.io，发布验收包含实际网络拉取测试。

```sh
docker compose pull crawler
docker compose run --rm crawler --help
docker compose run --rm crawler run
```

- 日常计划只执行已拉取、已固定的版本，不在每次抓取前自动更新镜像。容器替换不能删除 /data，下载新镜像也不能被当作重新初始化数据库的理由。
- 回退前检查旧代码能否读新 schema；兼容时改回旧 digest，不兼容时用升级前备份恢复并执行第十一节的远端版本核对，不能只换镜像后继续写旧备份。
